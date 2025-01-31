# COM activation for .NET Core on Windows

## Purpose

In order to more fully support the vast number of existing .NET Framework users in their transition to .NET Core, support of the COM activation scenario in .NET Core is required. Without this support it is not possible for many .NET Framework consumers to even consider transitioning to .NET Core. The intent of this document is to describe aspects of COM activation for a .NET class written for .NET Core. This support includes but is not limited to activation scenarios such as the [`CoCreateInstance()`](https://docs.microsoft.com/windows/desktop/api/combaseapi/nf-combaseapi-cocreateinstance) API in C/C++ or from within a [Windows Script Host](https://docs.microsoft.com/windows/desktop/com/using-com-objects-in-windows-script-host) instance.

COM activation in this document is currently limited to in-proc scenarios. Scenarios involving out-of-proc COM activation are deferred.

### Requirements

* Discover all installed versions of .NET Core.
* Load the appropriate version of .NET Core for the class if a .NET Core instance is not running, or validate the currently existing .NET Core instance can satisfy the class requirement.
* Return an [`IClassFactory`](https://docs.microsoft.com/windows/desktop/api/unknwnbase/nn-unknwnbase-iclassfactory) implementation that will construct an instance of the .NET class.
* Support the discrimination of concurrently loaded CLR versions.

### Environment matrix

The following list represents an exhaustive activation matrix.

| Server | Client | Current Support |
| --- | --- | :---: |
| COM* | .NET Core | Yes |
| .NET Core | COM* | Yes |
| .NET Core | .NET Core | Yes |
| .NET Framework | .NET Core | No |
| .NET Core | .NET Framework | No |

\* 'COM' is used to indicate any COM environment (e.g. C/C++) other than .NET.

## Design

One of the basic issues with the activation of a .NET class within a COM environment is the loading or discovery of an appropriate CLR instance. The .NET Framework addressed this issue through a system wide shim library (described below). The .NET Core scenario has different requirements and limitations on system impact and as such an identical solution may not be optimal or tenable.

### .NET Framework class COM activation

The .NET Framework uses a shim library (`mscoree.dll`) to facilitate the loading of the CLR into a process performing activation - one of the many uses of `mscoree.dll`. When .NET Framework 4.0 was released, `mscoreei.dll` was introduced to provide a level of indirection between the system installed shim (`mscoree.dll`) and a specific framework shim as well as to enable side-by-side CLR scenarios. An important consideration of the system wide shim is that of servicing. Servicing `mscoree.dll` is difficult since any process with a loaded .NET Framework instance will have the shim loaded, thus requiring a system reboot in order to service the shim.

During .NET class registration, the shim is identified as the in-proc server for the class. Additional metadata is inserted into the registry to indicate what .NET assembly to load and what type to activate. For example, in addition to the typical [in-proc server](https://docs.microsoft.com/windows/desktop/com/inprocserver32) registry values the following values are added to the registry for the `TypeLoadException` class.

```
"Assembly"="mscorlib, Version=1.0.5000.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"
"Class"="System.TypeLoadException"
"RuntimeVersion"="v1.1.4322"
```

The above registration is typically done with the [`RegAsm.exe`](https://docs.microsoft.com/dotnet/framework/tools/regasm-exe-assembly-registration-tool) tool. Alternatively, registry scripts can be generated by `RegAsm.exe`.

### .NET Core class COM activation

In .NET Core, our intent will be to avoid a system wide shim library. This decision may add additional cost for deployment scenarios, but will reduce servicing and engineering costs by making deployment more explicit and less magic.

The current .NET Core hosting solutions are described in detail at [Documentation/design-docs/host-components.md](https://github.com/dotnet/runtime/tree/main/docs/design/features/host-components.md). Along with the existing hosts an additional customizable COM activation host library (`comhost.dll`) will be added. This library (henceforth identified as 'shim') will export the required functions for COM class activation and registration and act in a way similar to .NET Framework's `mscoree.dll`.

>[`HRESULT DllGetClassObject(REFCLSID rclsid, REFIID riid, LPVOID *ppv);`](https://docs.microsoft.com/windows/desktop/api/combaseapi/nf-combaseapi-dllgetclassobject)

>[`HRESULT DllCanUnloadNow();`](https://docs.microsoft.com/windows/desktop/api/combaseapi/nf-combaseapi-dllcanunloadnow)

>[`HRESULT DllRegisterServer();`](https://msdn.microsoft.com/library/windows/desktop/ms682162(v=vs.85).aspx)

>[`HRESULT DllUnregisterServer();`](https://msdn.microsoft.com/library/windows/desktop/ms691457(v=vs.85).aspx)

When `DllGetClassObject()` is called in a COM activation scenario, the following steps will occur. The calling of `DllGetClassObject()` is usually accomplished through an implicit or explcit call to `CoCreateInstance()`.

1) Determine additional registration information needed for activation.
    * The shim will check for an embedded manifest. If the shim does not contain an embedded manifest, the shim will check if a file with the `<shim_name>.clsidmap` naming format exists adjacent to it. Build tooling handles shim customization, including renaming the shim to be based on the managed assembly's name (e.g. `NetComServer.dll` will have a custom shim called `NetComServer.comhost.dll`). If the shim is signed the shim will **not** attempt to discover the manifest on disk.
    * The manifest will contain a mapping from [`CLSID`](https://docs.microsoft.com/windows/desktop/com/com-class-objects-and-clsids) to managed assembly name and the [Fully-Qualified Name](https://docs.microsoft.com/dotnet/framework/reflection-and-codedom/specifying-fully-qualified-type-names) for the type. The format of this manifest is defined below. The shim's embedded mapping always takes precedence and in the case an embedded mapping is found, a `.clsidmap` file on disk will never be used.
    * The manifest will define an exhaustive list of .NET classes the shim is permitted to provide.
    * If a [`.runtimeconfig.json`](https://github.com/dotnet/cli/blob/master/Documentation/specs/runtime-configuration-file.md) file exists adjacent to the target managed assembly (`<assembly>.runtimeconfig.json`), that file is used to describe the target framework and CLR configuration. The documentation for the `.runtimeconfig.json` format defines under what circumstances this file may be optional.
1) The `DllGetClassObject()` function verifies the `CLSID` mapping has a mapping for the `CLSID`.
    * If the `CLSID` is unknown in the mapping the traditional `CLASS_E_CLASSNOTAVAILABLE` is returned.
1) The shim attempts to load the latest version of the `hostfxr` library and retrieves the `hostfxr_initialize_for_runtime_config()` and `hostfxr_get_runtime_delegate()` exports.
1) The target assembly name is computed by stripping off the `.comhost.dll` prefix and replacing it with `.dll`. Using the name of the target assembly, the path to the `.runtimeconfig.json` file is then computed.
1) The `hostfxr_initialize_for_runtime_config()` export is called.
1) Based on the `.runtimeconfig.json` the [framework](https://docs.microsoft.com/dotnet/core/packages#frameworks) to use can be determined and the appropriate `hostpolicy` library path is computed.
1) The `hostpolicy` library is loaded and various exports are retrieved.
    * If a `hostpolicy` instance is already loaded, the one presently loaded is re-used.
    * If a CLR is active within the process, the requested CLR version will be validated against that CLR. If version satisfiability fails, activation will fail.
1) The `corehost_load()` export is called to initialize `hostpolicy`.
    - Prior to .NET Core 3.0, during application activation the `corehost_load()` export would always initialize `hostpolicy` regardless if initialization had already been performed. For .NET Core 3.0, calling the function again will not re-initialize `hostpolicy`, but simply return.
1) The `hostfxr_get_runtime_delegate()` export is called
1) The `hostfxr_get_runtime_delegate()` export calls into `hostpolicy` and determines if the associated `coreclr` library has been loaded and if so, uses the existing activated CLR instance. If a CLR instance is not available, `hostpolicy` will load `coreclr` and activate a new CLR instance.
    * If a CLR is active within the process, the requested CLR version will be validated against that CLR. If version satisfiability fails, activation will fail.
1) A request to the CLR is made to create a managed delegate to a static "activation" method. The delegate is returned to the shim to attempt activation of the requested class.
    * The details of the activation API are implementation defined, but presently reside in `System.Private.CoreLib` on the `Internal.Runtime.InteropServices.ComActivator` class:
        ``` csharp
        [StructLayout(LayoutKind.Sequential)]
        [CLSCompliant(false)]
        public unsafe struct ComActivationContextInternal
        {
            public Guid ClassId;
            public Guid InterfaceId;
            public char* AssemblyPathBuffer;
            public char* AssemblyNameBuffer;
            public char* TypeNameBuffer;
            public IntPtr ClassFactoryDest;
        }

        public static class ComActivator
        {
            ...
            [CLSCompliant(false)]
            public static int GetClassFactoryForTypeInternal(ref ComActivationContextInternal cxtInt);
            ...
        }
        ```
        Note this API is not exposed outside of `System.Private.CoreLib` and is subject to change at any time.
    * The loading of the assembly will take place in a new [`AssemblyLoadContext`](https://docs.microsoft.com/dotnet/api/system.runtime.loader.assemblyloadcontext) for dependency isolation. Each assembly path will get a separate `AssemblyLoadContext`. This means that if an assembly provides multiple COM servers all of the servers from that assembly will reside in the same `AssemblyLoadContext`.
    * The created `AssemblyLoadContext` will use an [`AssemblyDependencyResolver`](https://github.com/dotnet/runtime/issues/27787) that was supplied with the path to the assembly to load assemblies.
1) The `IClassFactory` instance is returned to the caller of `DllGetClassObject()` to attempt class activation.

The `DllCanUnloadNow()` function will always return `S_FALSE` indicating the shim is never able to be unloaded. This matches .NET Framework semantics but may be adjusted in the future if needed.

The `DllRegisterServer()` and `DllUnregisterServer()` functions adhere to the [COM registration contract](https://docs.microsoft.com/windows/desktop/com/classes-and-servers) and enable registration and unregistration of the classes defined in the `CLSID` mapping manifest. Discovery of the mapping manifest is identical to that which occurs during a call to `DllGetClassObject()`.

#####  CLSID map format

The `CLSID` mapping manifest is a JSON format (`.clsidmap` extension when on disk) that defines a mapping from `CLSID` to an assembly name and type name tuple as well as an optional [ProgID](https://docs.microsoft.com/windows/win32/com/-progid--key). Each `CLSID` mapping is a key in the outer JSON object.

``` json
{
    "<clsid>": {
        "assembly": "<assembly_name>",
        "type": "<type_name>",
        "progid": "<prog_id>"
    }
}
```

### .NET Core COM server creation

1) A new .NET Core class library project is created using [`dotnet.exe`][dotnet_link].
1) A class is defined that has the [`GuidAttribute("<GUID>")`][guid_link] and the [`ComVisibleAttribute(true)`](https://docs.microsoft.com/dotnet/api/system.runtime.interopservices.comvisibleattribute).
    - In .NET Core, unlike .NET Framework, there is no generated class interface generation (i.e. `IClassX`). This means it is advantageous for users to have the class implement a marshalable interface.
    - A ProgID for the class can be defined using the [`ProgIdAttribute`](https://docs.microsoft.com/dotnet/api/system.runtime.interopservices.progidattribute). If a ProgID is not explicitly specified, the namespace and class name will be used as the ProgID. This follows the same semantics as .NET Framework COM servers.
1) The `EnableComHosting` property is added to the project file.
    - i.e. `<EnableComHosting>true</EnableComHosting>`
1) During class project build, the following actions occur if the `EnableComHosting` property is `true`:
    1) A `.runtimeconfig.json` file is created for the assembly.
    1) The resulting assembly is interrogated for classes with the attributes defined above and a `CLSID` map is created on disk (`.clsidmap`).
    1) The target Framework's shim binary (i.e. `comhost.dll`) is copied to the local output directory.
    1) The `comhost.dll` binary is renamed to `<assembly>.comhost.dll`.
    1) The generated `CLSID` map (`.clsidmap`) is embedded as a resource in the renamed `<assembly>.comhost.dll` binary.

### .NET Core COM server registration

Two options exist for registration and are a function of the intent of the class's author. The .NET Core platform will impose the deployment of a shim instance with a `.clsidmap` manifest. In order to address potential security concerns, the .NET Core tool chain by default will embed the `.clsidmap` in the customized shim. When the `.clsidmap` is embedded the customized shim allows for the implicit signing of the `.clsidmap` manifest. Once the shim is signed, the option for loading a non-embedded `.clsidmap` is disabled.

#### Registry

Class registration in the registry for .NET Core classes is greatly simplified and is now identical to that of a non-managed COM class. This is possible due to the presence of the aforementioned `.clsidmap` manifest. The application developer will be able to use the traditional [`regsvr32.exe`](https://docs.microsoft.com/windows-server/administration/windows-commands/regsvr32) tool for class registration.

#### Registration-Free

[RegFree COM for .NET](https://docs.microsoft.com/dotnet/framework/interop/configure-net-framework-based-com-components-for-reg) is another style of registration, but does not require registry access. This approach is complicated by the use of [application manifests](https://docs.microsoft.com/windows/desktop/SbsCs/application-manifests), but does have benefits for limiting environment impact and simplifying deployment. A severe limitation of this approach is that in order to use RegFree COM with a .NET class, the Window OS assumes the use of `mscoree.dll` for the in-proc server. Without a change in the Windows OS, this assumption in the RegFree .NET scenario makes the existing manifest approach a broken scenario for .NET Core.

An example of a RegFree manifest for a .NET Framework class is below - note the absence of specifying a hosting server library (i.e. `mscoree.dll` is implied for the `clrClass` element).

``` xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
    <assemblyIdentity
        type="win32"
        name="NetComServer"
        version="1.0.0.0" />

    <clrClass
        clsid="{3C58BBC9-3966-4B58-8EE2-398CBBC9FDC4}"
        name="NetComServer.Server"
        threadingModel="Both"
        runtimeVersion="v4.0.30319">
    </clrClass>
</assembly>
```

Due to the above issues with traditional RegFree manifests and .NET classes, an alternative system must be employed to enable a low-impact style of class registration for .NET Core.

The .NET Core steps for RegFree are as follows:

1) The native application will still define an application manifest, but instead of specifying the managed assembly as a dependency the application will define the shim as a dependent assembly.
    ``` xml
    <?xml version="1.0" encoding="utf-8" standalone="yes" ?>
    <assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
        <assemblyIdentity
            type="win32"
            name="COMClientPrimitives"
            version="1.0.0.0" />

        <dependency>
            <dependentAssembly>
                <!-- RegFree COM - CoreCLR Shim -->
                <assemblyIdentity
                    type="win32"
                    name="NetComServer.comhost.X"
                    version="1.0.0.0" />
            </dependentAssembly>
        </dependency>
    </assembly>
    ```
1) The tool chain can optionally generate a [SxS](https://docs.microsoft.com/windows/desktop/sbscs/about-side-by-side-assemblies-) manifest for the shim. Both the SxS manifest _and_ the shim library will need to be app-local for the scenario to work. Note that the application developer is responsible for adding to or merging the generated shim's manifest with one the user may have defined for other scenarios. An example shim manifest is defined below and with it the SxS logic will naturally know to query the shim for the desired class. Note that multiple `comClass` tags can be added.
    ``` xml
    <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
    <assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
        <assemblyIdentity
            type="win32"
            name="NetComServer.comhost.X"
            version="1.0.0.0" />

        <file name="NetComServer.comhost.dll">
            <!-- NetComServer.Server -->
            <comClass
                clsid="{3C58BBC9-3966-4B58-8EE2-398CBBC9FDC4}"
                threadingModel="Both" />
        </file>
    </assembly>
    ```
1) When the native application starts up, its SxS manifest will be read and dependency assemblies discovered. Exported COM classes will also be registered in the process.
1) COM activation then proceeds as defined above starting with a call to the shim's `DllGetClassObject()` export.

## Compatibility concerns

* Side-by-side concerns with the registration of classes that are defined in both .NET Framework and .NET Core.
    - i.e. Both classes have identical [`Guid`][guid_link] values.
* RegFree COM will not work the same between .NET Framework and .NET Core.
    - See details above.
* Servicing of the .NET Framework shim (`mscoree.dll`) was done at the system level. In the .NET Core scenario the onus is on the application developer to have a servicing process in place for the shim.
* There is no support for different versions of .NET Core running concurrently. This is not the case in .NET Framework where a 2.0 and 4.0 runtime can run in parallel. A potential future solution would be support for out-of-proc COM servers.

## References

[Calling COM Components from .NET Clients](https://msdn.microsoft.com/library/ms973800.aspx)

[Calling a .NET Component from a COM Component](https://msdn.microsoft.com/library/ms973802.aspx)

[Using COM Types in Managed Code](https://docs.microsoft.com/previous-versions/dotnet/netframework-4.0/3y76b69k%28v%3dvs.100%29)

[Exposing .NET Framework Components to COM](https://docs.microsoft.com/dotnet/framework/interop/exposing-dotnet-components-to-com)

<!-- Common links -->
[guid_link]: https://docs.microsoft.com/dotnet/api/system.runtime.interopservices.guidattribute
[dotnet_link]: https://docs.microsoft.com/dotnet/core/tools/dotnet
