# WasmEdge Go API Documentation

The followings are the guides to working with the WasmEdge-Go.

## Table of Contents

* [Getting Started](#Getting-Started)
  * [WasmEdge Installation](#WasmEdge-Installation)
  * [Get WasmEdge-go](#Get-WasmEdge-go)
  * [WasmEdge-go Extensions](#WasmEdge-go-Extensions)
* [WasmEdge-go Basics](#WasmEdge-go-Basics)
  * [Version](#Version)
  * [Logging Settings](#Logging-Settings)
  * [Value Types](#Value-Types)
  * [Results](#Results)
  * [Contexts And Their Life Cycles](#Contexts-And-Their-Life-Cycles)
  * [WASM data structures](#Wasm-data-structures)
  * [Configurations](#Configurations)
  * [Statistics](#Statistics)
* [WasmEdge VM](#WasmEdge-VM)
  * [WASM Execution Example With VM Context](#WASM-Execution-Example-With-VM-Context)
  * [VM Creations](#VM-Creations)
  * [Preregistrations](#Preregistrations)
  * [Host Module Registrations](#Host-Module-Registrations)
  * [WASM Registrations And Executions](#WASM-Registrations-And-Executions)
  * [Instance Tracing](#Instance-Tracing)
* [WasmEdge Runtime](#WasmEdge-Runtime)
  * [WASM Execution Example Step-By-Step](#WASM-Execution-Example-Step-By-Step)
  * [Loader](#Loader)
  * [Validator](#Validator)
  * [Executor](#Executor)
  * [AST Module](#AST-Module)
  * [Store](#Store)
  * [Instances](#Instances)
  * [Host Functions](#Host-Functions)
* [WasmEdge AOT Compiler](#WasmEdge-AOT-Compiler)
  * [Compilation Example](#Compilation-Example)
  * [Compiler Options](#Compiler-Options)

## Getting Started

The WasmEdge-go requires golang version >= 1.15. Please check your golang version before installation. Developers can [download golang here](https://golang.org/dl/).

```bash
$ go version
go version go1.16.5 linux/amd64
```

### WasmEdge Installation

Developers must [install the WasmEdge shared library](install.md) with the same `WasmEdge-go` release or pre-release version.

```bash
wget -qO- https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- -v 0.9.0-rc.3
```

For the developers need the `TensorFlow` or `Image` extension for `WasmEdge-go`, please install the `WasmEdge` with extensions:

```bash
wget -qO- https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- -e tf,image -v 0.9.0-rc.3
```

Noticed that the `TensorFlow` and `Image` extensions are only for the `Linux` platforms.
After installation, developers can use the `source` command to update the include and linking searching path.

### Get WasmEdge-go

After the WasmEdge installation, developers can get the `WasmEdge-go` package and build in your Go project directory.

```bash
$ go get github.com/second-state/WasmEdge-go/wasmedge@v0.9.0-rc3
$ go build
```

### WasmEdge-go Extensions

By default, the `WasmEdge-go` only turns on the basic runtime.

`WasmEdge-go` has the following extensions:

* Tensorflow
  * This extension supports the host functions in [WasmEdge-tensorflow](https://github.com/second-state/WasmEdge-tensorflow).
  * The `TensorFlow` extension when installing `WasmEdge` is required. Please install `WasmEdge` with the `-e tensorflow` command.
  * For using this extension, the tag `tensorflow` when building is required:

    ```bash
    $ go build -tags tensorflow
    ```

* Image
  * This extension supports the host functions in [WasmEdge-image](https://github.com/second-state/WasmEdge-image).
  * The `Image` extension when installing `WasmEdge` is required. Please install `WasmEdge` with the `-e image` command.
  * For using this extension, the tag `image` when building is required:

    ```bash
    $ go build -tags image
    ```

Users can also turn on the multiple extensions when building:

```bash
$ go build -tags image,tensorflow
```

For examples, please refer to the [example repository](https://github.com/second-state/WasmEdge-go-examples/).

## WasmEdge-go Basics

In this partition, we will introduce the utilities and concepts of WasmEdge-go APIs and data structures.

### Version

The `Version` related APIs provide developers to check for the installed WasmEdge shared library version.

```go
import "github.com/second-state/WasmEdge-go/wasmedge"

verstr := wasmedge.GetVersion() // Will be `string` of WasmEdge version.
vermajor := wasmedge.GetVersionMajor() // Will be `uint` of WasmEdge major version number.
verminor := wasmedge.GetVersionMinor() // Will be `uint` of WasmEdge minor version number.
verpatch := wasmedge.GetVersionPatch() // Will be `uint` of WasmEdge patch version number.
```

### Logging Settings

The `wasmedge.SetLogErrorLevel()` and `wasmedge.SetLogDebugLevel()` APIs can set the logging system to debug level or error level. By default, the error level is set, and the debug info is hidden.

### Value Types

In WasmEdge-go, the APIs will automatically do the convertion for the built-in types, and implement the data structure for the reference types.

1. Number types: `i32`, `i64`, `f32`, and `f64`

    * Convert the `uint32` and `int32` to `i32` automatically when passing a value into WASM.
    * Convert the `uint64` and `int64` to `i64` automatically when passing a value into WASM.
    * Convert the `uint` and `int` to `i32` automatically when passing a value into WASM in 32-bit system.
    * Convert the `uint` and `int` to `i64` automatically when passing a value into WASM in 64-bit system.
    * Convert the `float32` to `f32` automatically when passing a value into WASM.
    * Convert the `float64` to `f64` automatically when passing a value into WASM.
    * Convert the `i32` from WASM to `int32` when getting a result.
    * Convert the `i64` from WASM to `int64` when getting a result.
    * Convert the `f32` from WASM to `float32` when getting a result.
    * Convert the `f64` from WASM to `float64` when getting a result.

2. Number type: `v128` for the `SIMD` proposal

    Developers should use the `wasmedge.NewV128()` to generate a `v128` value, and use the `wasmedge.GetV128()` to get the value.

    ```go
    val := wasmedge.NewV128(uint64(1234), uint64(5678))
    high, low := val.GetVal()
    // `high` will be uint64(1234), `low` will be uint64(5678)
    ```

3. Reference types: `FuncRef` and `ExternRef` for the `Reference-Types` proposal

    ```go
    funcref := wasmedge.NewFuncRef(10)
    // Create a `FuncRef` with function index 10.

    num := 1234
    // `num` is a `int`.
    externref := wasmedge.NewExternRef(&num)
    // Create an `ExternRef` which reference to the `num`.
    num = 5678
    // Modify the `num` to 5678.
    numref := externref.GetRef().(*int)
    // Get the original reference from the `ExternRef`.
    fmt.Println(*numref)
    // Will print `5678`.
    numref.Release()
    // Should call the `Release` method.
    ```

### Results

The `Result` object specifies the execution status. Developers can use the `Error()` function to get the error message.

```go
// Assume that `vm` is a `wasmedge.VM` object.
res, err = vm.Execute(...) // Ignore the detail of parameters.
// Assume that `res, err` are the return values for executing a function with `vm`.
if err != nil {
    fmt.Println("Error message:", err.Error())
}
```

### Contexts And Their Life Cycles

The objects, such as `VM`, `Store`, and `Function`, are composed of `Context`s in the WasmEdge shared library.
All of the contexts can be created by calling the corresponding `New` APIs, and will be automatically deleted by the finalizers when the garbage collection is activated.
Developers can also call the corresponding `Release` functions of the contexts to forcely delete the object and release the resources immediately.
Noticed that it's not necessary to call the `Release` functions for the contexts which are retrieved from other contexts but not created from the `New` APIs.

```go
// Create a Configure.
conf := wasmedge.NewConfigure()
// Release the `conf` immediately.
conf.Release()
```

The details of other contexts will be introduced later.

### WASM Data Structures

The WASM data structures are used for creating instances or can be queried from instance contexts.
The details of instances creation will be introduced in the [Instances](#Instances).

1. Limit

    The `Limit` struct presents the minimum and maximum value data structure.

    ```go
    lim1 := wasmedge.NewLimit(12)
    fmt.Println(lim1.HasMax())
    // Will print `false`.
    fmt.Println(lim1.GetMin())
    // Will print `12`.

    lim2 := wasmedge.NewLimitWithMax(15, 50)
    fmt.Println(lim2.HasMax())
    // Will print `true`.
    fmt.Println(lim2.GetMin())
    // Will print `15`.
    fmt.Println(lim2.GetMax())
    // Will print `50`.
    ```

2. Function type context

    The `FunctionType` is an object holds the function type context and used for the `Function` creation, checking the value types of a `Function` instance, or getting the function type with function name from VM.
    Developers can use the `FunctionType` APIs to get the parameter or return value types information.

    ```go
    functype := wasmedge.NewFunctionType(
        []wasmedge.ValType{
            wasmedge.ValType_ExternRef,
            wasmedge.ValType_I32,
            wasmedge.ValType_I64,
        }, []wasmedge.ValType{
            wasmedge.ValType_F32,
            wasmedge.ValType_F64,
        })

    plen := functype.GetParametersLength()
    // `plen` will be 3.
    rlen := functype.GetReturnsLength()
    // `rlen` will be 2.
    plist := functype.GetParameters()
    // `plist` will be `[]wasmedge.ValType{wasmedge.ValType_ExternRef, wasmedge.ValType_I32, wasmedge.ValType_I64}`.
    rlist := functype.GetReturns()
    // `rlist` will be `[]wasmedge.ValType{wasmedge.ValType_F32, wasmedge.ValType_F64}`.

    functype.Release()
    ```

3. Table type context

    The `TableType` is an object holds the table type context and used for `Table` instance creation or getting information from `Table` instances.

    ```go
    lim := wasmedge.NewLimit(12)
    tabtype := wasmedge.NewTableType(wasmedge.RefType_ExternRef, lim)

    rtype := tabtype.GetRefType()
    // `rtype` will be `wasmedge.RefType_ExternRef`.
    getlim := tabtype.GetLimit()
    // `getlim` will be the same value as `lim`.

    tabtype.Release()
    ```

4. Memory type context

    The `MemoryType` is an object holds the memory type context and used for `Memory` instance creation or getting information from `Memory` instances.

    ```go
    lim := wasmedge.NewLimit(1)
    memtype := wasmedge.NewMemoryType(lim)

    getlim := memtype.GetLimit()
    // `getlim` will be the same value as `lim`.

    memtype.Release()
    ```

5. Global type context

    The `GlobalType` is an object holds the global type context and used for `Global` instance creation or getting information from `Global` instances.

    ```go
    globtype := wasmedge.NewGlobalType(wasmedge.ValType_F64, wasmedge.ValMut_Var)

    vtype := globtype.GetValType()
    // `vtype` will be `wasmedge.ValType_F64`.
    vmut := globtype.GetMutability()
    // `vmut` will be `wasmedge.ValMut_Var`.

    globtype.Release()
    ```

6. Import type context

    The `ImportType` is an object holds the import type context and used for getting the imports information from a [AST Module](#AST-Module).
    Developers can get the external type (`function`, `table`, `memory`, or `global`), import module name, and external name from an `ImportType` object.
    The details about querying `ImportType` objects will be introduced in the [AST Module](#AST-Module).

    ```go
    var ast *wasmedge.AST = ...
    // Assume that `ast` is returned by the `Loader` for the result of loading a WASM file.
    imptypelist := ast.ListImports()
    // Assume that `imptypelist` is an array listed from the `ast` for the imports.

    for i, imptype := range imptypelist {
        exttype := imptype.GetExternalType()
        // The `exttype` must be one of `wasmedge.ExternType_Function`, `wasmedge.ExternType_Table`,
        // wasmedge.ExternType_Memory`, or `wasmedge.ExternType_Global`.

        modname := imptype.GetModuleName()
        extname := imptype.GetExternalName()
        // Get the module name and external name of the imports.

        extval := imptype.GetExternalValue()
        // The `extval` is the type of `interface{}` which indicates one of `*wasmedge.FunctionType`,
        // `*wasmedge.TableType`, `*wasmedge.MemoryType`, or `*wasmedge.GlobalType`.
    }
    ```

7. Export type context

    The `ExportType` is an object holds the export type context is used for getting the exports information from a [AST Module](#AST-Module).
    Developers can get the external type (`function`, `table`, `memory`, or `global`) and external name from an `Export Type` context.
    The details about querying `ExportType` objects will be introduced in the [AST Module](#AST-Module).

    ```go
    var ast *wasmedge.AST = ...
    // Assume that `ast` is returned by the `Loader` for the result of loading a WASM file.
    exptypelist := ast.ListExports()
    // Assume that `exptypelist` is an array listed from the `ast` for the exports.

    for i, exptype := range exptypelist {
        exttype := exptype.GetExternalType()
        // The `exttype` must be one of `wasmedge.ExternType_Function`, `wasmedge.ExternType_Table`,
        // wasmedge.ExternType_Memory`, or `wasmedge.ExternType_Global`.

        extname := exptype.GetExternalName()
        // Get the external name of the exports.

        extval := exptype.GetExternalValue()
        // The `extval` is the type of `interface{}` which indicates one of `*wasmedge.FunctionType`,
        // `*wasmedge.TableType`, `*wasmedge.MemoryType`, or `*wasmedge.GlobalType`.
    }
    ```

## Go-API Examples

### Embed a function

Note: At this time, we require Rust compiler version 1.50 or less in order for WebAssembly functions to work with WasmEdge’s Golang API. We will [catch up to the latest Rust](https://github.com/WasmEdge/WasmEdge/issues/264) compiler version once the Interface Types spec is finalized and supported.

In [this example](https://github.com/second-state/WasmEdge-go-examples/tree/master/go_BindgenFuncs), we will demonstrate how to call a few simple WebAssembly functions from a Golang app. The [functions](https://github.com/second-state/WasmEdge-go-examples/blob/master/go_BindgenFuncs/rust_bindgen_funcs/src/lib.rs) are written in Rust, and require complex call parameters and return values. The #[wasm_bindgen] macro is needed for the compiler tools to auto-generate the correct code to pass call parameters from Golang to WebAssembly.

Note: The WebAssembly spec only supports a few simple data types out of the box. It [does not support](https://medium.com/wasm/strings-in-webassembly-wasm-57a05c1ea333) types such as string and array. In order to pass rich types in Golang to WebAssembly, the compiler needs to convert them to simple integers. For example, it converts a string into an integer memory address and an integer length. The wasm_bindgen tool, embedded in rustwasmc, does this conversion automatically.

```rust
use wasm_bindgen::prelude::*;
use num_integer::lcm;
use sha3::{Digest, Sha3_256, Keccak256};

#[wasm_bindgen]
pub fn say(s: &str) -> String {
  let r = String::from("hello ");
  return r + s;
}

#[wasm_bindgen]
pub fn obfusticate(s: String) -> String {
  (&s).chars().map(|c| {
    match c {
      'A' ..= 'M' | 'a' ..= 'm' => ((c as u8) + 13) as char,
      'N' ..= 'Z' | 'n' ..= 'z' => ((c as u8) - 13) as char,
      _ => c
    }
  }).collect()
}

#[wasm_bindgen]
pub fn lowest_common_multiple(a: i32, b: i32) -> i32 {
  let r = lcm(a, b);
  return r;
}

#[wasm_bindgen]
pub fn sha3_digest(v: Vec<u8>) -> Vec<u8> {
  return Sha3_256::digest(&v).as_slice().to_vec();
}

#[wasm_bindgen]
pub fn keccak_digest(s: &[u8]) -> Vec<u8> {
  return Keccak256::digest(s).as_slice().to_vec();
}
```

First, we use the rustwasmc tool to compile the Rust source code into WebAssembly bytecode functions using Rust 1.50 or less.

```bash
$ rustup default 1.50.0
$ cd rust_bindgen_funcs
$ rustwasmc build
# The output WASM will be pkg/rust_bindgen_funcs_lib_bg.wasm
```

The [Golang source code](https://github.com/second-state/WasmEdge-go-examples/blob/master/go_BindgenFuncs/bindgen_funcs.go) to run the WebAssembly function in WasmEdge is as follows. The ExecuteBindgen() function calls the WebAssembly function and passes the call parameters using the #[wasm_bindgen] convention.


```go
package main

import (
    "fmt"
    "os"
    "github.com/second-state/WasmEdge-go/wasmedge"
)

func main() {
    /// Expected Args[0]: program name (./bindgen_funcs)
    /// Expected Args[1]: wasm or wasm-so file (rust_bindgen_funcs_lib_bg.wasm))

    wasmedge.SetLogErrorLevel()

    var conf = wasmedge.NewConfigure(wasmedge.WASI)
    var vm = wasmedge.NewVMWithConfig(conf)
    var wasi = vm.GetImportObject(wasmedge.WASI)
    wasi.InitWasi(
        os.Args[1:],     /// The args
        os.Environ(),    /// The envs
        []string{".:."}, /// The mapping directories
        []string{},      /// The preopens will be empty
    )

    /// Instantiate wasm
    vm.LoadWasmFile(os.Args[1])
    vm.Validate()
    vm.Instantiate()

    /// Run bindgen functions
    var res interface{}
    var err error
    
    res, err = vm.ExecuteBindgen("say", wasmedge.Bindgen_return_array, []byte("bindgen funcs test"))
    if err == nil {
        fmt.Println("Run bindgen -- say:", string(res.([]byte)))
    } 
    res, err = vm.ExecuteBindgen("obfusticate", wasmedge.Bindgen_return_array, []byte("A quick brown fox jumps over the lazy dog"))
    if err == nil {
        fmt.Println("Run bindgen -- obfusticate:", string(res.([]byte)))
    } 
    res, err = vm.ExecuteBindgen("lowest_common_multiple", wasmedge.Bindgen_return_i32, int32(123), int32(2))
    if err == nil {
        fmt.Println("Run bindgen -- lowest_common_multiple:", res.(int32))
    } 
    res, err = vm.ExecuteBindgen("sha3_digest", wasmedge.Bindgen_return_array, []byte("This is an important message"))
    if err == nil {
        fmt.Println("Run bindgen -- sha3_digest:", res.([]byte))
    } 
    res, err = vm.ExecuteBindgen("keccak_digest", wasmedge.Bindgen_return_array, []byte("This is an important message"))
    if err == nil {
        fmt.Println("Run bindgen -- keccak_digest:", res.([]byte))
    } 

    vm.Release()
    conf.Release()
}
```

Next, let's build the Golang application with the WasmEdge Golang SDK.

```bash
$ go get github.com/second-state/WasmEdge-go/wasmedge@v0.9.0-rc3
$ go build
```

Run the Golang application and it will run the WebAssembly functions embedded in the WasmEdge runtime.

```bash
$ ./bindgen_funcs rust_bindgen_funcs/pkg/rust_bindgen_funcs_lib_bg.wasm
Run bindgen -- say: hello bindgen funcs test
Run bindgen -- obfusticate: N dhvpx oebja sbk whzcf bire gur ynml qbt
Run bindgen -- lowest_common_multiple: 246
Run bindgen -- sha3_digest: [87 27 231 209 189 105 251 49 159 10 211 250 15 159 154 181 43 218 26 141 56 199 25 45 60 10 20 163 54 211 195 203]
Run bindgen -- keccak_digest: [126 194 241 200 151 116 227 33 216 99 159 22 107 3 177 169 216 191 114 156 174 193 32 159 246 228 245 133 52 75 55 27]
```

### Embed a full program

Note: You can use the latest Rust compiler to create a standalone WasmEdge application with a main.rs and then embed it into a Golang application.

Besides functions, the WasmEdge Golang SDK can also [embed standalone WebAssembly applications](https://github.com/second-state/WasmEdge-go-examples/tree/master/go_ReadFile) — ie a Rust application with a main() function compiled into WebAssembly.

Our [demo Rust application](https://github.com/second-state/WasmEdge-go-examples/tree/master/go_ReadFile/rust_readfile) reads from a file. Note that there is no need for #{wasm_bindgen] here, as the WebAssembly program’s input and output data are now passed by the STDIN and STDOUT.


```rust
use std::env;
use std::fs::File;
use std::io::{self, BufRead};

fn main() {
    // Get the argv.
    let args: Vec<String> = env::args().collect();
    if args.len() <= 1 {
        println!("Rust: ERROR - No input file name.");
        return;
    }

    // Open the file.
    println!("Rust: Opening input file \"{}\"...", args[1]);
    let file = match File::open(&args[1]) {
        Err(why) => {
            println!("Rust: ERROR - Open file \"{}\" failed: {}", args[1], why);
            return;
        },
        Ok(file) => file,
    };

    // Read lines.
    let reader = io::BufReader::new(file);
    let mut texts:Vec<String> = Vec::new();
    for line in reader.lines() {
        if let Ok(text) = line {
            texts.push(text);
        }
    }
    println!("Rust: Read input file \"{}\" succeeded.", args[1]);

    // Get stdin to print lines.
    println!("Rust: Please input the line number to print the line of file.");
    let stdin = io::stdin();
    for line in stdin.lock().lines() {
        let input = line.unwrap();
        match input.parse::<usize>() {
            Ok(n) => if n > 0 && n <= texts.len() {
                println!("{}", texts[n - 1]);
            } else {
                println!("Rust: ERROR - Line \"{}\" is out of range.", n);
            },
            Err(e) => println!("Rust: ERROR - Input \"{}\" is not an integer: {}", input, e),
        }
    }
    println!("Rust: Process end.");
}
```

Use the rustwasmc tool to compile the application into WebAssembly.

```bash
$ cd rust_readfile
$ rustwasmc build
# The output file will be pkg/rust_readfile.wasm
```

The Golang source code to run the WebAssembly function in WasmEdge is as follows.

```go
package main

import (
    "os"
    "github.com/second-state/WasmEdge-go/wasmedge"
)

func main() {
    wasmedge.SetLogErrorLevel()

    var conf = wasmedge.NewConfigure(wasmedge.REFERENCE_TYPES)
    conf.AddConfig(wasmedge.WASI)
    var vm = wasmedge.NewVMWithConfig(conf)
    var wasi = vm.GetImportObject(wasmedge.WASI)
    wasi.InitWasi(
        os.Args[1:],     /// The args
        os.Environ(),    /// The envs
        []string{".:."}, /// The mapping directories
        []string{},      /// The preopens will be empty
    )

    /// Instantiate wasm. _start refers to the main() function
    vm.RunWasmFile(os.Args[1], "_start")

    vm.Release()
    conf.Release()
}
```

Next, let's build the Golang application with the WasmEdge Golang SDK.

```bash
$ go get github.com/second-state/WasmEdge-go/wasmedge@v0.9.0-rc3
$ go build
```

Run the Golang application.

```bash
$ ./read_file rust_readfile/pkg/rust_readfile.wasm file.txt
Rust: Opening input file "file.txt"...
Rust: Read input file "file.txt" succeeded.
Rust: Please input the line number to print the line of file.
# Input "5" and press Enter.
5
# The output will be the 5th line of `file.txt`:
abcDEF___!@#$%^
# To terminate the program, send the EOF (Ctrl + D).
^D
# The output will print the terminate message:
Rust: Process end.
```

More examples can be found at [the WasmEdge-go-examples GitHub repo.](https://github.com/second-state/WasmEdge-go-examples)

### WasmEdge AOT Compiler in GO

The [go_WasmAOT example](https://github.com/second-state/WasmEdge-go-examples/tree/master/go_WasmAOT) provide a tool for compiling a WASM file into compiled-WASM for AOT mode.
