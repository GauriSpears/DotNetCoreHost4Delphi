# Integrando Delphi com .NET Core: Guia Completo

## Introdução

Este guia apresenta abordagens para integrar aplicações Delphi com o .NET Core, focando no método de hosting que permite chamar código gerenciado .NET Core a partir de código nativo Delphi.

## Contexto do Problema

O .NET Framework permitia facilmente chamar objetos e métodos a partir de aplicações nativas como Delphi:
- Via registro COM no Windows (opção limitada ao Windows)
- Via processo HOST inicializado pelo código nativo (mais flexível)

Com a transição do .NET Framework (agora legado) para o .NET Core (atual e futuro), surgiram novos desafios:
- O .NET Core continua suportando COM apenas no Windows
- É necessária uma estratégia multiplataforma para integração nativa

## Soluções Disponíveis

### 1. Nova API de Hosting do .NET Core

A Microsoft introduziu uma nova API de hosting para o .NET Core a partir da versão 3.0, que substitui a antiga coreclrhost.h. Esta nova solução usa:

- **Biblioteca nethost**: fornece funções para localizar a biblioteca hostfxr
- **Biblioteca hostfxr**: contém as APIs para inicializar e gerenciar o runtime do .NET Core

Eis o fluxo básico para usar esta API:

1. Usar `get_hostfxr_path()` para localizar o hostfxr
2. Carregar a biblioteca hostfxr e obter seus pontos de exportação
3. Inicializar o runtime com `hostfxr_initialize_for_runtime_config()`
4. Obter um delegate para funcionalidades do runtime com `hostfxr_get_runtime_delegate()`
5. Usar o delegate para carregar assemblies e obter ponteiros para métodos gerenciados
6. Chamar os métodos gerenciados

### 2. Biblioteca dotNetCore4Delphi

Uma solução comercial produzida pela CrystalNet Technologies, especificamente para integração entre Delphi e .NET Core:

- Hospeda o Common Language Runtime (CoreCLR) diretamente do Delphi
- Permite carregar e acessar assemblies/tipos de bibliotecas .NET Core
- Fornece mecanismos para invocar membros de tipos .NET Core
- Inclui suporte para criar instâncias, gerenciar eventos e exceções
- Oferece ferramentas para gerar código Delphi a partir de bibliotecas .NET Core

## Implementação da Solução de Hosting

A abordagem recomendada para seu caso é a implementação nativa do hosting do .NET Core usando as bibliotecas nethost e hostfxr, seguindo o exemplo da Microsoft adaptado para Delphi.

### Passo 1: Configurar o Projeto

1. Crie um projeto Delphi
2. Adicione as declarações e mapeamentos para as APIs de hosting do .NET Core

### Passo 2: Declarar as Estruturas e APIs necessárias

Primeiro, você precisa criar mapeamentos Delphi para as estruturas e funções do nethost e hostfxr. Isto inclui:

```delphi
// Adaptar cabeçalhos do nethost.h
type
  char_t = WideChar;  // No Windows
  
  // Estrutura de parâmetros para get_hostfxr_path
  get_hostfxr_parameters = record
    size: SIZE_T;
    assembly_path: PWideChar;
    dotnet_root: PWideChar;
  end;
  Pget_hostfxr_parameters = ^get_hostfxr_parameters;

// Declarar a função da nethost.dll
function get_hostfxr_path(buffer: PWideChar; buffer_size: PSIZE_T; parameters: Pget_hostfxr_parameters): Integer; stdcall; external 'nethost.dll';
```

Em seguida, declare as estruturas e funções da hostfxr:

```delphi
// Tipo para os handles do hostfxr
type
  hostfxr_handle = Pointer;
  Phostfxr_handle = ^hostfxr_handle;
  
  // Tipos de delegates para funcionalidades do runtime
  hostfxr_delegate_type = (
    hdt_com_activation,
    hdt_load_in_memory_assembly,
    hdt_winrt_activation,
    hdt_com_register,
    hdt_com_unregister,
    hdt_load_assembly_and_get_function_pointer
  );

// Funções essenciais do hostfxr
type
  hostfxr_initialize_for_runtime_config_fn = function(runtime_config_path: PWideChar; parameters: Pointer; host_context_handle: Phostfxr_handle): Integer; stdcall;
  
  hostfxr_get_runtime_delegate_fn = function(host_context_handle: hostfxr_handle; r_type: hostfxr_delegate_type; delegate: PPointer): Integer; stdcall;
  
  hostfxr_close_fn = function(host_context_handle: hostfxr_handle): Integer; stdcall;
```

### 2. Implementação Delphi da API de Hosting

Para o lado Delphi, precisamos implementar o código que carrega o runtime .NET Core e comunica-se com nossa biblioteca bridge:

```delphi
unit NetCoreHost;

interface

uses
  System.SysUtils, Winapi.Windows;

const
  // Retornos da API
  Success                            = 0;
  Success_HostAlreadyInitialized     = 1;
  HostInvalidState                   = 2;
  CoreHostIncompatibleConfig         = 0x80008207;
  CoreHostLibLoadFailure             = 0x80008098;
  
  // Tipos de delegate
  HDT_COM_ACTIVATION                      = 0;
  HDT_LOAD_IN_MEMORY_ASSEMBLY            = 1;
  HDT_WINRT_ACTIVATION                   = 2;
  HDT_COM_REGISTER                       = 3;
  HDT_COM_UNREGISTER                     = 4;
  HDT_LOAD_ASSEMBLY_AND_GET_FUNCTION_POINTER = 5;
  HDT_GET_FUNCTION_POINTER               = 6;

type
  // Tipos para manipulação da API .NET Core
  THostfxrHandle = Pointer;
  PHostfxrHandle = ^THostfxrHandle;

  // Tipos para as funções importadas de hostfxr.dll
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

  // Tipo do delegate para carregar assembly e obter função
  TLoadAssemblyAndGetFunctionPointerFn = function(
    assembly_path: PWideChar;
    type_name: PWideChar;
    method_name: PWideChar;
    delegate_type_name: PWideChar;
    reserved: Pointer;
    delegate: PPointer): Integer; stdcall;

  // Tipo para os métodos da biblioteca bridge
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

  // Classe principal para hosting do .NET Core
  TNetCoreHost = class
  private
    FHostfxrHandle: THostfxrHandle;
    FHostfxrLib: HMODULE;
    FNetstandardDll: string;
    FBridgeDll: string;
    FRuntimeConfigPath: string;
    
    // Ponteiros para as funções importadas
    FInitializeFn: THostfxrInitializeForRuntimeConfigFn;
    FGetDelegateFn: THostfxrGetRuntimeDelegateFn;
    FCloseFn: THostfxrCloseFn;
    
    // Ponteiro para o delegate do runtime
    FLoadAssemblyFn: TLoadAssemblyAndGetFunctionPointerFn;
    
    // Ponteiros para funções da bridge
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

// Função da nethost.dll
function get_hostfxr_path(buffer: PWideChar; buffer_size: PSIZE_T; parameters: Pointer): Integer; stdcall; external 'nethost.dll';

{ TNetCoreHost }

constructor TNetCoreHost.Create(const NetCoreRuntimePath: string);
begin
  inherited Create;
  
  FHostfxrLib := 0;
  FHostfxrHandle := nil;
  
  // Caminho do runtime deve ser passado pelo usuário
  // Exemplo: C:\Program Files\dotnet\shared\Microsoft.NETCore.App\6.0.9
  FNetstandardDll := TPath.Combine(NetCoreRuntimePath, 'DelphiBridge.dll');
  FRuntimeConfigPath := TPath.Combine(ExtractFilePath(FNetstandardDll), 'DelphiBridge.runtimeconfig.json');
end;

destructor TNetCoreHost.Destroy;
begin
  // Limpar os recursos do runtime
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
  
  // Obter o caminho para hostfxr.dll
  buffer_size := Length(buffer);
  if get_hostfxr_path(@buffer[0], @buffer_size, nil) <> 0 then
    Exit;
    
  hostfxr_path := buffer;
  
  // Carregar a biblioteca hostfxr
  FHostfxrLib := LoadLibrary(PChar(hostfxr_path));
  if FHostfxrLib = 0 then
    Exit;
    
  // Obter os ponteiros para as funções
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
  // Inicializar o runtime com o arquivo de configuração
  rc := FInitializeFn(PWideChar(FRuntimeConfigPath), nil, @FHostfxrHandle);
  
  Result := (rc = Success) or (rc = Success_HostAlreadyInitialized);
end;

function TNetCoreHost.GetRuntimeDelegate: Boolean;
var
  rc: Integer;
  delegate: Pointer;
begin
  Result := False;
  
  // Obter o delegate para carregar assemblies
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
  
  // Carregar a função CreateInstance da bridge
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
  
  // Carregar a função InvokeMethod da bridge
  invokeMethodPtr := nil;
  rc := FLoadAssemblyFn(
    PWideChar(FNetstandardDll),
    PWideChar('DelphiBridge.Bridge, DelphiBridge'),
    PWideChar('InvokeMethod'),
    nil,  // Sem tipo de delegate específico - usando o padrão
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
    nil,  // Sem argumentos
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

Essa implementação é mais robusta que o exemplo anterior e fornece uma base sólida para integrar Delphi com .NET Core.


### Passo 4: Inicializar o Runtime e Carregar Assembly

```delphi
// Tipo para o delegate que carrega assemblies e obtém ponteiros de função
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
  
  // Inicializar o runtime
  cxt := nil;
  rc := init_fptr(PWideChar(config_path), nil, @cxt);
  if (rc <> 0) or (cxt = nil) then
  begin
    // Manipulação de erro
    close_fptr(cxt);
    Exit;
  end;
  
  // Obter o delegate para carregar assemblies
  load_assembly_and_get_function_pointer := nil;
  rc := get_delegate_fptr(cxt, hdt_load_assembly_and_get_function_pointer, 
                         @load_assembly_and_get_function_pointer);
  
  if (rc <> 0) or (load_assembly_and_get_function_pointer = nil) then
  begin
    // Manipulação de erro
    close_fptr(cxt);
    Exit;
  end;
  
  Result := load_assembly_and_get_function_pointer_fn(load_assembly_and_get_function_pointer);
  close_fptr(cxt);
end;
```

### Passo 5: Chamada de Método Gerenciado

```delphi
// Exemplo de estrutura para passar argumentos
type
  lib_args = record
    message: PWideChar;
    number: Integer;
  end;
  
// Tipo para o método gerenciado a ser chamado
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
  // Caminho para o arquivo de configuração runtime do .NET
  config_path := 'MeuComponente.runtimeconfig.json';
  
  // Obter o delegate para carregar assemblies
  load_assembly := GetDotNetLoadAssembly(config_path);
  if load_assembly = nil then
    Exit;
  
  // Configurar os caminhos e nomes
  assembly_path := 'MeuComponente.dll';
  type_name := 'MeuComponente.Classe, MeuComponente';
  method_name := 'MetodoExemplo';
  
  // Carregar o assembly e obter o ponteiro para o método
  hello := nil;
  rc := load_assembly(PWideChar(assembly_path), PWideChar(type_name), PWideChar(method_name), 
                     nil, nil, @hello);
  
  if (rc <> 0) or (hello = nil) then
    Exit;
  
  // Preparar argumentos e chamar o método
  args.message := 'Olá do Delphi!';
  args.number := 42;
  
  rc := hello(@args, SizeOf(args));
  
  ShowMessage('Resultado da chamada: ' + IntToStr(rc));
end;
```

## Componentes da Solução Proposta

### 1. Biblioteca de Bridge em C#

Esta biblioteca atua como a interface de alto nível para o Delphi, abstraindo a complexidade das chamadas entre os ambientes.

```csharp
using System;
using System.Reflection;
using System.Runtime.InteropServices;
using System.Collections.Generic;

namespace DelphiBridge
{
    // Classe principal exposta para Delphi
    public static class Bridge
    {
        // Cache de objetos para referência por ID
        private static Dictionary<int, object> _objectCache = new Dictionary<int, object>();
        private static int _nextObjectId = 1;
        
        // Cache de tipos para referência rápida
        private static Dictionary<string, Type> _typeCache = new Dictionary<string, Type>();
        
        // Para funções que retornam objetos
        private static Dictionary<int, object> _resultCache = new Dictionary<int, object>();
        private static int _nextResultId = 1;

        // Método chamado pelo Delphi para criar uma instância
        [UnmanagedCallersOnly]
        public static int CreateInstance(IntPtr assemblyNamePtr, IntPtr typeNamePtr, IntPtr argsPtr, int argsCount)
        {
            try
            {
                string assemblyName = Marshal.PtrToStringUTF8(assemblyNamePtr);
                string typeName = Marshal.PtrToStringUTF8(typeNamePtr);
                
                // Localizar/carregar o tipo
                Type type = GetTypeFromName(assemblyName, typeName);
                if (type == null)
                    return -1;
                
                // Criar instância (com ou sem parâmetros)
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
                
                // Armazenar e retornar ID
                return RegisterObject(instance);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error in CreateInstance: {ex.Message}");
                return -1;
            }
        }
        
        // Chamada de método
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
                
                // Invocar método
                object result = type.InvokeMember(
                    methodName, 
                    BindingFlags.Instance | BindingFlags.Public | BindingFlags.InvokeMethod, 
                    null, instance, args);
                
                // Armazenar resultado se necessário
                if (result != null && resultPtr != IntPtr.Zero)
                {
                    int resultId = RegisterResult(result);
                    Marshal.WriteInt32(resultPtr, resultId);
                    return 1; // Sucesso com resultado
                }
                
                return 0; // Sucesso sem resultado
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error in InvokeMethod: {ex.Message}");
                return -1;
            }
        }
        
        // Método auxiliar para registrar objetos
        private static int RegisterObject(object obj)
        {
            int id = _nextObjectId++;
            _objectCache[id] = obj;
            return id;
        }
        
        // Método auxiliar para registrar resultados
        private static int RegisterResult(object result)
        {
            int id = _nextResultId++;
            _resultCache[id] = result;
            return id;
        }
        
        // Desserialização de argumentos
        private static object[] DeserializeArgs(IntPtr argsPtr, int count)
        {
            // Implementação simplificada - em uma versão real,
            // teria código para desserializar diferentes tipos de argumentos
            object[] result = new object[count];
            
            // Exemplo simplificado para inteiros
            for (int i = 0; i < count; i++)
            {
                result[i] = Marshal.ReadInt32(argsPtr, i * sizeof(int));
            }
            
            return result;
        }
        
        // Helper para localizar tipos
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

## Considerações Importantes

1. **Compatibilidade de Versões**: Certifique-se de que o .NET Core usado na biblioteca está disponível no sistema ou é distribuído com sua aplicação.

2. **runtimeconfig.json**: Este arquivo é essencial para inicializar o runtime correto. Para bibliotecas, adicione `<GenerateRuntimeConfigurationFiles>True</GenerateRuntimeConfigurationFiles>` ao arquivo .csproj.

3. **Gerenciamento de Memória**: Tenha cuidado com a passagem de strings e outros objetos complexos entre Delphi e .NET Core.

4. **Ciclo de Vida dos Objetos**: Por padrão, o exemplo acima usa métodos estáticos. Para trabalhar com instâncias, você precisará implementar um sistema de rastreamento de referências entre os ambientes.

5. **Multiplataforma**: A implementação mostrada funciona em Windows. Para Linux e macOS, são necessárias adaptações nas declarações de tipos e carregamento de bibliotecas.

## Exemplo de Uso da Implementação Proposta

Para facilitar o uso da nossa solução, podemos criar uma camada de abstração de alto nível que simplifica consideravelmente a interação com o .NET Core:

```delphi
unit NetCoreObjects;

interface

uses
  System.SysUtils, System.Generics.Collections, System.TypInfo, NetCoreHost;

type
  // Classe base para todas as classes que encapsulam objetos .NET
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

  // Classe base para factory de proxies .NET
  TNetCoreProxyFactory = class
  private
    FNetCoreHost: TNetCoreHost;
    FAssemblyName: string;
    FTypeName: string;
  public
    constructor Create(ANetCoreHost: TNetCoreHost; const AAssemblyName, ATypeName: string);
    
    function CreateInstance: TNetCoreObject; virtual;
  end;

// Código para uma classe específica (como exemplo):
type
  // Proxy para a classe Calculadora do .NET
  TCalculadora = class(TNetCoreObject)
  public
    function Somar(A, B: Integer): Integer;
    function Subtrair(A, B: Integer): Integer;
    function Multiplicar(A, B: Integer): Integer;
    function Dividir(A, B: Double): Double;
  end;

  // Factory para Calculadora
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
  // Aqui seria ideal ter um método para liberar o objeto no lado .NET
  // mas usaremos o garbage collector dele por enquanto
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
  // Serializar argumentos - implementação simplificada
  // Na prática, você precisaria implementar a conversão entre Variant do Delphi
  // e os tipos que .NET espera
  
  ArgsPtr := nil;
  ArgsCount := Length(Args);
  
  // Invocar o método
  ResultValue := 0;
  FNetCoreHost.InvokeMethod(FObjectId, MethodName, ArgsPtr, ArgsCount, @ResultValue);
  
  // Converter o ResultValue para o valor adequado
  // Aqui também precisaria de implementação para converter tipos
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

### Exemplo de Uso:

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
    // Inicializar o host .NET Core
    Host := TNetCoreHost.Create('C:\Program Files\dotnet\shared\Microsoft.NETCore.App\6.0.9');
    try
      if not Host.Initialize then
      begin
        Writeln('Erro ao inicializar o runtime .NET Core');
        Exit;
      end;
      
      // Criar a factory
      CalcFactory := TCalculadoraFactory.Create(Host);
      try
        // Criar e usar a instância da calculadora
        Calculadora := CalcFactory.CreateInstance;
        try
          Resultado := Calculadora.Somar(10, 32);
          Writeln('Resultado da soma: ', Resultado);
          
          Resultado := Calculadora.Multiplicar(5, 7);
          Writeln('Resultado da multiplicação: ', Resultado);
        finally
          Calculadora.Free;
        end;
      finally
        CalcFactory.Free;
      end;
    finally
      Host.Free;
    end;
    
    Writeln('Pressione Enter para sair...');
    Readln;
  except
    on E: Exception do
      Writeln(E.ClassName, ': ', E.Message);
  end;
end.
```

Esta abordagem orientada a objetos fornece uma interface Delphi limpa para trabalhar com objetos .NET, escondendo a complexidade da infraestrutura de interoperabilidade.

## Análise das Implementações Existentes

Após avaliar o código disponível, identifiquei três abordagens principais:

### 1. DDNRuntime (Solução Chinesa)

Esta implementação parece ser bastante completa e utiliza diretamente as APIs nativas do .NET Core:

- Usa a biblioteca coreclr.dll diretamente, chamando funções como `coreclr_initialize` e `coreclr_create_delegate`
- Gerencia o ciclo de vida do runtime .NET Core
- Implementa mapeamento de tipos entre Delphi e .NET Core
- Suporta chamadas de métodos, propriedades e eventos

A arquitetura é robusta, porém complexa, utilizando um DLL intermediário (NETCoreCLR.dll) para facilitar a comunicação.

### 2. dotNetCore4Delphi (Solução Comercial)

Esta implementação oferece uma API mais elegante e de alto nível:

- Encapsula as complexidades da interação com o .NET Core
- Usa a API nethost/hostfxr para localizar e inicializar o runtime
- Implementa mapeamento de tipos e injeção de dependência
- Oferece ferramentas de geração de código para facilitar a integração

A arquitetura é mais modular e orientada a objetos, mas o licenciamento por instalação torna inviável para seu caso de uso.

### 3. API de Hosting Direta da Microsoft

Esta é a abordagem mais fundamental, utilizando diretamente:

- `nethost.h` para localizar o hostfxr
- `hostfxr.h` para inicializar o runtime e obter delegates
- `coreclr_delegates.h` para trabalhar com funções remotas

## Conclusão e Recomendações

A integração entre Delphi e .NET Core é viável através da API de hosting nativa ou bibliotecas como as analisadas acima. A abordagem mais adequada para cenários com muitas estações de trabalho parece ser desenvolver uma solução própria baseada nos princípios da API de hosting da Microsoft, mas com algumas lições das implementações existentes.

Recomendado uma estratégia em duas partes:

1. **Componente Delphi**: Implementar um wrapper para a API de hosting do .NET Core que cuida do carregamento do runtime, localização de assemblies e chamada de métodos básicos.

2. **Biblioteca Helper em C#**: Criar uma biblioteca em .NET Core que serve como uma "ponte" facilitadora, expondo uma API simplificada para o Delphi e lidando com a reflexão, gerenciamento de objetos, e conversões de tipos no lado .NET.

Esta abordagem equilibra a necessidade de controle total da solução (evitando dependências externas com licenciamento complexo) com a facilidade de desenvolvimento, concentrando o código mais complexo no lado C# onde é mais fácil trabalhar com reflexão.

## Recursos Adicionais

- [Documentação oficial da Microsoft sobre hosting do .NET Core](https://learn.microsoft.com/pt-br/dotnet/core/tutorials/netcore-hosting)
- [Repositório de exemplos em C++ da Microsoft](https://github.com/dotnet/samples/tree/main/core/hosting)