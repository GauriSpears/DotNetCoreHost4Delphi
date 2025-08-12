# Integrating Delphi with .NET Core: Complete Guide

## Introduction

This guide presents approaches for integrating Delphi applications with .NET Core, focusing on the hosting method that allows you to call managed .NET Core code from native Delphi code.

## Problem Context

The .NET Framework allowed you to easily call objects and methods from native applications like Delphi:
- Via COM registration on Windows (option limited to Windows)
- Via a host process initialized by native code (more flexible)

With the transition from .NET Framework (now legacy) to .NET Core (current and future), new challenges emerged:
- .NET Core continues to support COM only on Windows
- A cross-platform strategy is required for native integration

## Available Solutions

### 1. New .NET Core Hosting API

Microsoft introduced a new hosting API for .NET Core starting with version 3.0, which replaces the old coreclrhost.h. This new solution uses:

- **nethost library**: Provides functions to locate the hostfxr library
- **hostfxr library**: Contains the APIs for initializing and managing the .NET Core runtime

Here's the basic flow for using this API:

1. Use `get_hostfxr_path()` to locate hostfxr
2. Load the hostfxr library and get its export points
3. Initialize the runtime with `hostfxr_initialize_for_runtime_config()`
4. Get a delegate for runtime functionality with `hostfxr_get_runtime_delegate()`
5. Use the delegate to load assemblies and get pointers to managed methods
6. Call the managed methods

### 2. dotNetCore4Delphi Library

A commercial solution produced by CrystalNet Technologies, specifically for integration between Delphi and .NET Core:

- Hosts the Common Language Runtime (CoreCLR) directly from Delphi
- Allows you to load and access assemblies/types from .NET Core libraries
- Provides mechanisms for invoking members of .NET Core types
- Includes support for creating instances, managing events, and exceptions
- Provides tools for generating Delphi code from .NET Core libraries

## Implementing the Hosting Solution

The recommended approach for your case is a native implementation of .NET Core hosting using the nethost and hostfxr libraries, following the Microsoft example adapted for Delphi.

### Step 1: Configure the Project

1. Create a Delphi project
2. Add the declarations and mappings for the .NET Core hosting APIs

### Step 2: Declare the Required Structures and APIs

First, you need to create Delphi mappings for the nethost and hostfxr structures and functions. This includes:

```delphi
// Adapt headers of nethost.h
type
  char_t = WideChar;  // No Windows
  
  // Parameter structure for get_hostfxr_path
  get_hostfxr_parameters = record
    size: SIZE_T;
    assembly_path: PWideChar;
    dotnet_root: PWideChar;
  end;
  Pget_hostfxr_parameters = ^get_hostfxr_parameters;

// Declare the function of nethost.dll
function get_hostfxr_path(buffer: PWideChar; buffer_size: PSIZE_T; parameters: Pget_hostfxr_parameters): Integer; stdcall; external 'nethost.dll';
```

Next, declare the hostfxr structures and functions:

```delphi
// Type for the handles of the hostfxr
type
  hostfxr_handle = Pointer;
  Phostfxr_handle = ^hostfxr_handle;
  
  // Delegate types for runtime functionality
  hostfxr_delegate_type = (
    hdt_com_activation,
    hdt_load_in_memory_assembly,
    hdt_winrt_activation,
    hdt_com_register,
    hdt_com_unregister,
    hdt_load_assembly_and_get_function_pointer
  );

// Essential functions of hostfxr
type
  hostfxr_initialize_for_runtime_config_fn = function(runtime_config_path: PWideChar; parameters: Pointer; host_context_handle: Phostfxr_handle): Integer; stdcall;
  
  hostfxr_get_runtime_delegate_fn = function(host_context_handle: hostfxr_handle; r_type: hostfxr_delegate_type; delegate: PPointer): Integer; stdcall;
  
  hostfxr_close_fn = function(host_context_handle: hostfxr_handle): Integer; stdcall;
```

### 2. Delphi Implementation of the Hosting API

For the Delphi side, we need to implement the code that loads the .NET Core runtime and communicates with our bridge library:

```delphi
unit NetCoreHost;

interface

uses
  System.SysUtils, Winapi.Windows;

const
  // API Returns
  Success                            = 0;
  Success_HostAlreadyInitialized     = 1;
  HostInvalidState                   = 2;
  CoreHostIncompatibleConfig         = 0x80008207;
  CoreHostLibLoadFailure             = 0x80008098;
  
  // Types of delegate
  HDT_COM_ACTIVATION                      = 0;
  HDT_LOAD_IN_MEMORY_ASSEMBLY            = 1;
  HDT_WINRT_ACTIVATION                   = 2;
  HDT_COM_REGISTER                       = 3;
  HDT_COM_UNREGISTER                     = 4;
  HDT_LOAD_ASSEMBLY_AND_GET_FUNCTION_POINTER = 5;
  HDT_GET_FUNCTION_POINTER               = 6;

type
  // Types for .NET Core API manipulation
  THostfxrHandle = Pointer;
  PHostfxrHandle = ^THostfxrHandle;

  // Types for functions imported from hostfxr.dll
  THostfxrInitializeForRuntimeConfigFn = function(
    runtime_config_path: PWideChar;
    parameters: Pointer;
    host_context_handle: PHostfxrHandle): Integer; stdcall;
    
  THostfxrGetRuntimeDelegateFn = function(
    host_context_handle: THostfxrHandle;
    delegate_type: Integer;
    delegate: PPointer): Integer; stdcall;
    
  THostfxrCloseFn = function(
    host_context_handle: THostfxrHandle): Integer; stdcall;

  // Delegate type to load assembly and get function
  TLoadAssemblyAndGetFunctionPointerFn = function(
    assembly_path: PWideChar;
    type_name: PWideChar;
    method_name: PWideChar;
    delegate_type_name: PWideChar;
    reserved: Pointer;
    delegate: PPointer): Integer; stdcall;

  // Type for bridge library methods
  TCreateInstanceFn = function(
    assemblyName: PWideChar;
    typeName: PWideChar;
    args: Pointer;
    argsCount: Integer): Integer; stdcall;
    
  TInvokeMethodFn = function(
    objectId: Integer; 
    methodName: PWideChar;
    args: Pointer;
    argsCount: Integer;
    resultPtr: Pointer): Integer; stdcall;

  // Core class for .NET Core hosting
  TNetCoreHost = class
  private
    FHostfxrHandle: THostfxrHandle;
    FHostfxrLib: HMODULE;
    FNetstandardDll: string;
    FBridgeDll: string;
    FRuntimeConfigPath: string;
    
    // Pointers to imported functions
    FInitializeFn: THostfxrInitializeForRuntimeConfigFn;
    FGetDelegateFn: THostfxrGetRuntimeDelegateFn;
    FCloseFn: THostfxrCloseFn;
    
    // Pointer to the runtime delegate
    FLoadAssemblyFn: TLoadAssemblyAndGetFunctionPointerFn;
    
    // Pointers to bridge functions
    FCreateInstanceFn: TCreateInstanceFn;
    FInvokeMethodFn: TInvokeMethodFn;
    
    function LoadHostfxr: Boolean;
    function InitializeRuntime: Boolean;
    function GetRuntimeDelegate: Boolean;
    function LoadBridgeAssembly: Boolean;
  public
    constructor Create(const NetCoreRuntimePath: string);
    destructor Destroy; override;
    
    function Initialize: Boolean;
    
    function CreateInstance(const AssemblyName, TypeName: string): Integer;
    function InvokeMethod(ObjectId: Integer; const MethodName: string; Args: Pointer; 
      ArgsCount: Integer; ResultPtr: Pointer): Integer;
  end;

implementation

uses
  System.IOUtils;

// Function of nethost.dll
function get_hostfxr_path(buffer: PWideChar; buffer_size: PSIZE_T; parameters: Pointer): Integer; stdcall; external 'nethost.dll';

{ TNetCoreHost }

constructor TNetCoreHost.Create(const NetCoreRuntimePath: string);
begin
  inherited Create;
  
  FHostfxrLib := 0;
  FHostfxrHandle := nil;
  
  // Runtime path must be passed by user
  // Example: C:\Program Files\dotnet\shared\Microsoft.NETCore.App\6.0.9
  FNetstandardDll := TPath.Combine(NetCoreRuntimePath, 'DelphiBridge.dll');
  FRuntimeConfigPath := TPath.Combine(ExtractFilePath(FNetstandardDll), 'DelphiBridge.runtimeconfig.json');
end;

destructor TNetCoreHost.Destroy;
begin
  // Clean up runtime resources
  if FHostfxrHandle <> nil then
    FCloseFn(FHostfxrHandle);
    
  if FHostfxrLib <> 0 then
    FreeLibrary(FHostfxrLib);
    
  inherited;
end;

function TNetCoreHost.Initialize: Boolean;
begin
  Result := LoadHostfxr and InitializeRuntime and GetRuntimeDelegate and LoadBridgeAssembly;
end;

function TNetCoreHost.LoadHostfxr: Boolean;
var
  buffer: array[0..MAX_PATH] of WideChar;
  buffer_size: SIZE_T;
  hostfxr_path: string;
begin
  Result := False;
  
  // Get the path to hostfxr.dll
  buffer_size := Length(buffer);
  if get_hostfxr_path(@buffer[0], @buffer_size, nil) <> 0 then
    Exit;
    
  hostfxr_path := buffer;
  
  // Load the hostfxr library
  FHostfxrLib := LoadLibrary(PChar(hostfxr_path));
  if FHostfxrLib = 0 then
    Exit;
    
  // Get pointers to functions
  FInitializeFn := GetProcAddress(FHostfxrLib, 'hostfxr_initialize_for_runtime_config');
  FGetDelegateFn := GetProcAddress(FHostfxrLib, 'hostfxr_get_runtime_delegate');
  FCloseFn := GetProcAddress(FHostfxrLib, 'hostfxr_close');
  
  if (@FInitializeFn = nil) or (@FGetDelegateFn = nil) or (@FCloseFn = nil) then
    Exit;
    
  Result := True;
end;

function TNetCoreHost.InitializeRuntime: Boolean;
var
  rc: Integer;
begin
  // Initialize the runtime with the configuration file
  rc := FInitializeFn(PWideChar(FRuntimeConfigPath), nil, @FHostfxrHandle);
  
  Result := (rc = Success) or (rc = Success_HostAlreadyInitialized);
end;

function TNetCoreHost.GetRuntimeDelegate: Boolean;
var
  rc: Integer;
  delegate: Pointer;
begin
  Result := False;
  
  // Get the delegate to load assemblies
  delegate := nil;
  rc := FGetDelegateFn(FHostfxrHandle, HDT_LOAD_ASSEMBLY_AND_GET_FUNCTION_POINTER, @delegate);
  
  if (rc <> Success) or (delegate = nil) then
    Exit;
    
  FLoadAssemblyFn := delegate;
  Result := True;
end;

function TNetCoreHost.LoadBridgeAssembly: Boolean;
var
  rc: Integer;
  createInstancePtr, invokeMethodPtr: Pointer;
begin
  Result := False;
  
  // Load the bridge's CreateInstance function
  createInstancePtr := nil;
  rc := FLoadAssemblyFn(
    PWideChar(FNetstandardDll),
    PWideChar('DelphiBridge.Bridge, DelphiBridge'),
    PWideChar('CreateInstance'),
    nil,  // Sem tipo de delegate específico - usando o padrão
    nil,
    @createInstancePtr);
    
  if (rc <> Success) or (createInstancePtr = nil) then
    Exit;
    
  FCreateInstanceFn := createInstancePtr;
  
  // Load the bridge's InvokeMethod function
  invokeMethodPtr := nil;
  rc := FLoadAssemblyFn(
    PWideChar(FNetstandardDll),
    PWideChar('DelphiBridge.Bridge, DelphiBridge'),
    PWideChar('InvokeMethod'),
    nil,  // No specific delegate type - using default
    nil,
    @invokeMethodPtr);
    
  if (rc <> Success) or (invokeMethodPtr = nil) then
    Exit;
    
  FInvokeMethodFn := invokeMethodPtr;
  
  Result := True;
end;

function TNetCoreHost.CreateInstance(const AssemblyName, TypeName: string): Integer;
begin
  if @FCreateInstanceFn = nil then
    Exit(-1);
    
  Result := FCreateInstanceFn(
    PWideChar(AssemblyName),
    PWideChar(TypeName),
    nil,  // No arguments
    0);
end;

function TNetCoreHost.InvokeMethod(ObjectId: Integer; const MethodName: string; 
  Args: Pointer; ArgsCount: Integer; ResultPtr: Pointer): Integer;
begin
  if @FInvokeMethodFn = nil then
    Exit(-1);
    
  Result := FInvokeMethodFn(
    ObjectId,
    PWideChar(MethodName),
    Args,
    ArgsCount,
    ResultPtr);
end;

end.
```

This implementation is more robust than the previous example and provides a solid foundation for integrating Delphi with .NET Core.


### Step 4: Initialize the Runtime and Load Assembly

```delphi
// Type for delegate that loads assemblies and gets function pointers
type
  load_assembly_and_get_function_pointer_fn = function(
    assembly_path: PWideChar;
    type_name: PWideChar;
    method_name: PWideChar;
    delegate_type_name: PWideChar;
    reserved: Pointer;
    delegate: PPointer): Integer; stdcall;

function GetDotNetLoadAssembly(config_path: string): load_assembly_and_get_function_pointer_fn;
var
  cxt: hostfxr_handle;
  rc: Integer;
  load_assembly_and_get_function_pointer: Pointer;
begin
  Result := nil;
  
  // Initialize the runtime
  cxt := nil;
  rc := init_fptr(PWideChar(config_path), nil, @cxt);
  if (rc <> 0) or (cxt = nil) then
  begin
    // Error handling
    close_fptr(cxt);
    Exit;
  end;
  
  // Get the delegate to load assemblies
  load_assembly_and_get_function_pointer := nil;
  rc := get_delegate_fptr(cxt, hdt_load_assembly_and_get_function_pointer, 
                         @load_assembly_and_get_function_pointer);
  
  if (rc <> 0) or (load_assembly_and_get_function_pointer = nil) then
  begin
    // Error handling
    close_fptr(cxt);
    Exit;
  end;
  
  Result := load_assembly_and_get_function_pointer_fn(load_assembly_and_get_function_pointer);
  close_fptr(cxt);
end;
```

### Step 5: Managed Method Call

```delphi
// Example of a structure for passing arguments
type
  lib_args = record
    message: PWideChar;
    number: Integer;
  end;
  
// Type for the managed method to be called
type
  component_entry_point_fn = function(args: Pointer; size_bytes: Integer): Integer; stdcall;

procedure ExecuteDotNetMethod;
var
  load_assembly: load_assembly_and_get_function_pointer_fn;
  hello: component_entry_point_fn;
  args: lib_args;
  assembly_path, type_name, method_name: string;
  config_path: string;
  rc: Integer;
begin
  // Path to the .NET runtime configuration file
  config_path := 'MeuComponente.runtimeconfig.json';
  
  // Get the delegate to load assemblies
  load_assembly := GetDotNetLoadAssembly(config_path);
  if load_assembly = nil then
    Exit;
  
  // Configure paths and names
  assembly_path := 'MeuComponente.dll';
  type_name := 'MeuComponente.Classe, MeuComponente';
  method_name := 'MetodoExemplo';
  
  // Load the assembly and get the pointer to the method
  hello := nil;
  rc := load_assembly(PWideChar(assembly_path), PWideChar(type_name), PWideChar(method_name), 
                     nil, nil, @hello);
  
  if (rc <> 0) or (hello = nil) then
    Exit;
  
  // Prepare arguments and call the method
  args.message := 'Hello from Delphi!';
  args.number := 42;
  
  rc := hello(@args, SizeOf(args));
  
  ShowMessage('Call result: ' + IntToStr(rc));
end;
```

## Components of the Proposed Solution

### 1. Bridge Library in C#

This library acts as the high-level interface to Delphi, abstracting away the complexity of calls between environments.

```csharp
using System;
using System.Reflection;
using System.Runtime.InteropServices;
using System.Collections.Generic;

namespace DelphiBridge
{
    // Main class exposed to Delphi
    public static class Bridge
    {
        // Object cache for reference by ID
        private static Dictionary<int, object> _objectCache = new Dictionary<int, object>();
        private static int _nextObjectId = 1;
        
        // Type cache for quick reference
        private static Dictionary<string, Type> _typeCache = new Dictionary<string, Type>();
        
        // For functions that return objects
        private static Dictionary<int, object> _resultCache = new Dictionary<int, object>();
        private static int _nextResultId = 1;

        // Method called by Delphi to create an instance
        [UnmanagedCallersOnly]
        public static int CreateInstance(IntPtr assemblyNamePtr, IntPtr typeNamePtr, IntPtr argsPtr, int argsCount)
        {
            try
            {
                string assemblyName = Marshal.PtrToStringUTF8(assemblyNamePtr);
                string typeName = Marshal.PtrToStringUTF8(typeNamePtr);
                
                // Find/Load Type
                Type type = GetTypeFromName(assemblyName, typeName);
                if (type == null)
                    return -1;
                
                // Create instance (with or without parameters)
                object instance;
                if (argsCount > 0 && argsPtr != IntPtr.Zero)
                {
                    object[] args = DeserializeArgs(argsPtr, argsCount);
                    instance = Activator.CreateInstance(type, args);
                }
                else
                {
                    instance = Activator.CreateInstance(type);
                }
                
                // Store and return ID
                return RegisterObject(instance);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error in CreateInstance: {ex.Message}");
                return -1;
            }
        }
        
        // Method call
        [UnmanagedCallersOnly]
        public static int InvokeMethod(int objectId, IntPtr methodNamePtr, IntPtr argsPtr, int argsCount, IntPtr resultPtr)
        {
            try
            {
                if (!_objectCache.TryGetValue(objectId, out object instance))
                    return -1;
                
                string methodName = Marshal.PtrToStringUTF8(methodNamePtr);
                
                // Localizar método
                Type type = instance.GetType();
                object[] args = null;
                
                if (argsCount > 0 && argsPtr != IntPtr.Zero)
                    args = DeserializeArgs(argsPtr, argsCount);
                
                // Invoke method
                object result = type.InvokeMember(
                    methodName, 
                    BindingFlags.Instance | BindingFlags.Public | BindingFlags.InvokeMethod, 
                    null, instance, args);
                
                // Store result if necessary
                if (result != null && resultPtr != IntPtr.Zero)
                {
                    int resultId = RegisterResult(result);
                    Marshal.WriteInt32(resultPtr, resultId);
                    return 1; // Success with results
                }
                
                return 0; // Success without result
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error in InvokeMethod: {ex.Message}");
                return -1;
            }
        }
        
        // Helper method for registering objects
        private static int RegisterObject(object obj)
        {
            int id = _nextObjectId++;
            _objectCache[id] = obj;
            return id;
        }
        
        // Auxiliary method for recording results
        private static int RegisterResult(object result)
        {
            int id = _nextResultId++;
            _resultCache[id] = result;
            return id;
        }
        
        // Argument deserialization
        private static object[] DeserializeArgs(IntPtr argsPtr, int count)
        {
            // Simplified implementation - in a real version,
            // there would be code to deserialize different types of arguments
            object[] result = new object[count];
            
            // Simplified example for integers
            for (int i = 0; i < count; i++)
            {
                result[i] = Marshal.ReadInt32(argsPtr, i * sizeof(int));
            }
            
            return result;
        }
        
        // Helper to locate types
        private static Type GetTypeFromName(string assemblyName, string typeName)
        {
            string key = $"{assemblyName}|{typeName}";
            
            if (_typeCache.TryGetValue(key, out Type cachedType))
                return cachedType;
            
            try
            {
                Assembly assembly = Assembly.Load(assemblyName);
                Type type = assembly.GetType(typeName, true);
                _typeCache[key] = type;
                return type;
            }
            catch
            {
                return null;
            }
        }
    }
}
```

## Important Considerations

1. **Version Compatibility**: Make sure the .NET Core version used in the library is available on your system or distributed with your application.

2. **runtimeconfig.json**: This file is essential for initializing the correct runtime. For libraries, add `<GenerateRuntimeConfigurationFiles>True</GenerateRuntimeConfigurationFiles>` to the .csproj file.

3. **Memory Management**: Be careful when passing strings and other complex objects between Delphi and .NET Core.

4. **Object Lifecycle**: By default, the example above uses static methods. To work with instances, you will need to implement a reference tracking system between environments.

5. **Multiplatform**: The implementation shown works on Windows. For Linux and macOS, adaptations to type declarations and library loading are necessary.

## Example of Using the Proposed Implementation

To make our solution easier to use, we can create a high-level abstraction layer that significantly simplifies interaction with .NET Core:

```delphi
unit NetCoreObjects;

interface

uses
  System.SysUtils, System.Generics.Collections, System.TypInfo, NetCoreHost;

type
  // Base class for all classes that encapsulate .NET objects
  TNetCoreObject = class
  private
    FObjectId: Integer;
    FNetCoreHost: TNetCoreHost;
  protected
    function GetObjectId: Integer;
  public
    constructor Create(ANetCoreHost: TNetCoreHost; AObjectId: Integer); virtual;
    destructor Destroy; override;
    
    function InvokeMethod(const MethodName: string; Args: array of Variant): Variant;
    property ObjectId: Integer read GetObjectId;
  end;

  // Base class for .NET proxy factory
  TNetCoreProxyFactory = class
  private
    FNetCoreHost: TNetCoreHost;
    FAssemblyName: string;
    FTypeName: string;
  public
    constructor Create(ANetCoreHost: TNetCoreHost; const AAssemblyName, ATypeName: string);
    
    function CreateInstance: TNetCoreObject; virtual;
  end;

// Code for a specific class (as an example):
type
  // Proxy for the .NET Calculator class
  TCalculadora = class(TNetCoreObject)
  public
    function Somar(A, B: Integer): Integer;
    function Subtrair(A, B: Integer): Integer;
    function Multiplicar(A, B: Integer): Integer;
    function Dividir(A, B: Double): Double;
  end;

  // Factory for Calculator
  TCalculadoraFactory = class(TNetCoreProxyFactory)
  public
    constructor Create(ANetCoreHost: TNetCoreHost);
    function CreateInstance: TCalculadora; reintroduce;
  end;

implementation

{ TNetCoreObject }

constructor TNetCoreObject.Create(ANetCoreHost: TNetCoreHost; AObjectId: Integer);
begin
  inherited Create;
  FNetCoreHost := ANetCoreHost;
  FObjectId := AObjectId;
end;

destructor TNetCoreObject.Destroy;
begin
  // Here, it would be ideal to have a method to release the object on the .NET side.
  // but we'll use its garbage collector for now.
  inherited;
end;

function TNetCoreObject.GetObjectId: Integer;
begin
  Result := FObjectId;
end;

function TNetCoreObject.InvokeMethod(const MethodName: string; Args: array of Variant): Variant;
var
  ResultValue: Integer;
  ArgsPtr: Pointer;
  ArgsCount: Integer;
begin
  // Serialize arguments - simplified implementation
  // In practice, you would need to implement the conversion between Delphi Variant
  // and the types that .NET expects
  
  ArgsPtr := nil;
  ArgsCount := Length(Args);
  
  // Invoke the method
  ResultValue := 0;
  FNetCoreHost.InvokeMethod(FObjectId, MethodName, ArgsPtr, ArgsCount, @ResultValue);
  
  // Convert the ResultValue to the appropriate value
  // Here you would also need an implementation to convert types
  Result := ResultValue;
end;

{ TNetCoreProxyFactory }

constructor TNetCoreProxyFactory.Create(ANetCoreHost: TNetCoreHost; const AAssemblyName, ATypeName: string);
begin
  inherited Create;
  FNetCoreHost := ANetCoreHost;
  FAssemblyName := AAssemblyName;
  FTypeName := ATypeName;
end;

function TNetCoreProxyFactory.CreateInstance: TNetCoreObject;
var
  ObjectId: Integer;
begin
  ObjectId := FNetCoreHost.CreateInstance(FAssemblyName, FTypeName);
  if ObjectId <= 0 then
    raise Exception.Create('Erro ao criar instância de objeto .NET');
    
  Result := TNetCoreObject.Create(FNetCoreHost, ObjectId);
end;

{ TCalculadora }

function TCalculadora.Somar(A, B: Integer): Integer;
begin
  Result := InvokeMethod('Somar', [A, B]);
end;

function TCalculadora.Subtrair(A, B: Integer): Integer;
begin
  Result := InvokeMethod('Subtrair', [A, B]);
end;

function TCalculadora.Multiplicar(A, B: Integer): Integer;
begin
  Result := InvokeMethod('Multiplicar', [A, B]);
end;

function TCalculadora.Dividir(A, B: Double): Double;
begin
  Result := InvokeMethod('Dividir', [A, B]);
end;

{ TCalculadoraFactory }

constructor TCalculadoraFactory.Create(ANetCoreHost: TNetCoreHost);
begin
  inherited Create(ANetCoreHost, 'MinhaBiblioteca', 'MinhaBiblioteca.Calculadora');
end;

function TCalculadoraFactory.CreateInstance: TCalculadora;
var
  BaseObject: TNetCoreObject;
begin
  BaseObject := inherited CreateInstance;
  Result := TCalculadora.Create(FNetCoreHost, BaseObject.ObjectId);
  BaseObject.Free; // Liberar o objeto base, mas manter o ID
end;

end.
```

### Usage Example:

```delphi
program ExemploCoreIntegracao;

{$APPTYPE CONSOLE}

{$R *.res}

uses
  System.SysUtils,
  NetCoreHost,
  NetCoreObjects;

var
  Host: TNetCoreHost;
  CalcFactory: TCalculadoraFactory;
  Calculadora: TCalculadora;
  Resultado: Integer;

begin
  try
    // Initialize the .NET Core host
    Host := TNetCoreHost.Create('C:\Program Files\dotnet\shared\Microsoft.NETCore.App\6.0.9');
    try
      if not Host.Initialize then
      begin
        Writeln('Erro ao inicializar o runtime .NET Core');
        Exit;
      end;
      
      // Create the factory
      CalcFactory := TCalculadoraFactory.Create(Host);
      try
        // Create and use the calculator instance
        Calculadora := CalcFactory.CreateInstance;
        try
          Resultado := Calculadora.Somar(10, 32);
          Writeln('Result of the sum: ', Resultado);
          
          Resultado := Calculadora.Multiplicar(5, 7);
          Writeln('Multiplication result: ', Resultado);
        finally
          Calculadora.Free;
        end;
      finally
        CalcFactory.Free;
      end;
    finally
      Host.Free;
    end;
    
    Writeln('Press Enter to exit...');
    Readln;
  except
    on E: Exception do
      Writeln(E.ClassName, ': ', E.Message);
  end;
end.
```

This object-oriented approach provides a clean Delphi interface for working with .NET objects, hiding the complexity of the interoperability infrastructure.

## Analysis of Existing Implementations

After evaluating the available code, I identified three main approaches:

### 1. DDNRuntime (Chinese Solution)

This implementation appears to be quite complete and directly uses the native .NET Core APIs:

- Uses the coreclr.dll library directly, calling functions such as `coreclr_initialize` and `coreclr_create_delegate`
- Manages the .NET Core runtime lifecycle
- Implements type mapping between Delphi and .NET Core
- Supports method, property, and event calls

The architecture is robust but complex, using an intermediate DLL (NETCoreCLR.dll) to facilitate communication.

### 2. dotNetCore4Delphi (Commercial Solution)

This implementation offers a more elegant and high-level API:

- Encapsulates the complexities of interacting with .NET Core
- Uses the nethost/hostfxr API to locate and initialize the runtime
- Implements type mapping and dependency injection
- Offers code generation tools to facilitate integration

The architecture is more modular and object-oriented, but per-install licensing makes it impractical for your use case.

### 3. Microsoft Direct Hosting API

This is the most fundamental approach, directly using:

- `nethost.h` to locate hostfxr
- `hostfxr.h` to initialize the runtime and obtain delegates
- `coreclr_delegates.h` to work with remote functions

## Conclusion and Recommendations

Integration between Delphi and .NET Core is feasible through the native hosting API or libraries like those discussed above. The most appropriate approach for scenarios with many workstations seems to be to develop a custom solution based on the principles of Microsoft's hosting API, but with some lessons learned from existing implementations.

A two-part strategy is recommended:

1. Delphi Component: Implement a wrapper for the .NET Core hosting API that handles runtime loading, assemblies location, and basic method calling.

2. C# Helper Library: Create a .NET Core library that serves as a facilitating "bridge," exposing a simplified API to Delphi and handling reflection, object management, and type conversions on the .NET side.

This approach balances the need for complete control of the solution (avoiding external dependencies with complex licensing) with ease of development, concentrating the most complex code on the C# side where it is easier to work with reflection.

## Additional Resources

- [Documentação oficial da Microsoft sobre hosting do .NET Core](https://learn.microsoft.com/pt-br/dotnet/core/tutorials/netcore-hosting)
- [Repositório de exemplos em C++ da Microsoft](https://github.com/dotnet/samples/tree/main/core/hosting)
