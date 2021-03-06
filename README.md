---
title: LensVM Specification
tags: lens-vm, specification, draft
---

<p align="center">
<img src="https://i.imgur.com/u29eC7t.png">
</p>

# LensVM
![](https://img.shields.io/badge/Owner-@jsimnz-blue) ![](https://img.shields.io/badge/Status-DRAFT-yellow)

LensVM is a data migration/transformation engine powered by bi-directional lenses. This work is based on the [Cambria]() project by Ink&Switch. LensVM uses bi-directional lenses, written in WebAssembly for portability to transform data from one shape to another. This project houses a collection of specifications and implementations to realize the full goal of LensVM. This project relies on [Content Identified]() data models, [JSON-Schema](), and (maybe) [Ceramic Network]() for data consistency and schema registration.

LensVM design is heavily influenced by [proxy-wasm]() which is a WebAssembly based module/plugin framework for extending network proxy behaviour and functionality. 

Included is a ABI Specification for interactions between the host environment running LensVM, and the wasm module. Language specific SDKs enable easier development of ABI compliant modules. Lastly, host runtimes for executing LensVM are provided.

Module Languages
- [ ] Go
- [ ] AssemblyScript
- [ ] Rust
- [ ] C++

Host Runtimes
- [ ] Go
- [ ] JavaScript (Browser)
- [ ] Node
- [ ] Rust

## Motivation
Dealing with schemas and data shapes from multiple different sources, applications, or versions; centralized or decentralized, can become a nightmare of interopability as schemas quickly diverge from one another. This is seen in database migrations, application version evolution, and platform interactions. Time and time again two systems become completely independant from one another with little option for reconciliation without data loss on one or both sides.

## Goals
The goal for LensVM is to create a consistent system, that can operate in both centralized and decentralized environments, which allows developers to easily transform data from one structure to another, without mass coordination, and using a standards-first approach. 

The first implementation will allow bi-directional lenses to be compiled to WebAssembly from several origin languages like C++, Rust, and Go, and execute these lenses in any host environment that supports the WASI interface to succesfully transform data defined using JSON-Schema from one schema to another.

## Bi-Directional Lenses
Bi-Directional lenses that power LensVM are based on the work by the [Cambria]() project from [Ink & Switch] industrial research lab. You can read their essay on their implementaion and goals [here](). To summarize, these lenses are succinct transformations which can be applied in a forward and reverse direction to modify JSON objects field by field from one schema to another. This includes renaming fields, moving fields from sub objects, mapping fields from one type to another (e.g. scalar to array), and more.

These lenses operate in an isolated layer, with minimal additional meta-data to reduce complexity and dependancy on other elements of the software stack. 

Each lenses executes a single transformation, which can then be composed into more complex lenses, or into a array of lenses to be executed.

The original implementation of lenses from the [Cambria]() project operated on [JSON-Patch]() objects, and not the raw JSON objects. There are various reasons they chose this path, however LensVM lenses are defined to operate on the raw JSON objects directly. Our reason for this is to support lenses which depend on multiple fields at once. 

If for example we had an object like so:
```json
{
    "firstName": "John",
    "lastName": "Smith"
}
```

and we wanted to convert to a single `name` field, using the JSON-Patch approach would be unable to do so. Instead, LensVM would simply define a `combine` or `concat` lens which takes an array of fields, and a target field, like so:
```json
{
    "concat": {
       "source": [
           "firstName",
           "lastName"
       ],
       "destination": "name"
    }
}

```

![](https://i.imgur.com/wbhKoLx.png)
###### *image credit: https://www.inkandswitch.com/cambria.html*

## Composability & Execution
The main purpose of LensVM is to freely allow lenses to be written in one language, and used in any other language via WASM. Moreover, lens design is conducted in an open environment by developers in a bottom up grass roots community, akin to Ethereum EIP proposals. 

Lenses are accompanied by their matchin meta-data which defines their functional goal, parameter types, and dependancies on other lenses. All compiled lenses and meta-data are stored in a Content Identified data model like [IPLD](). This allows developers to pin-point a specific instance or version of a lens, and never have that dependancy broken by future updates.

LensVM uses language specific SDKs to create lens functions to ensure consistency and utility between languages, and host VM's. This SDK will provide hooks to import lenses from other langauges via the Lens Module and compiled WASM code. It will provide consistent utility functions to manage incoming data and parameters, and translate that to a Lens File.

## Module File (module.json/yaml)
A Lens Module is a combination of the compiled lens in WASM, and its associated meta-data into a single CID based object. A lens definition only exports a single transformation (like `rename`), however that exported function can itself depend on other pre-existing lenses, identified by the CIDs of their respective Lens Module files. These definitions can then be importent individually or as a group defined as a Lens Package.

Here's an example `Module File`

```json
{
    "name": "rename",
    "description": "Rename a field from source to destination",
    "url": "https://github.com/lens-vm/community-modules"
    
    "arguments": {
        "type": "object",
        "properties": {
            "source": {
                "description": "The source field for renaming",
                "type": "string"
            },
            "destination": {
                "description": "The destination field for renaming",
                "type": "string"
            }
        }
    },
    "import": {
        "extract": "Qm999"
    }
    
    "runtime": "wasm",
    "language": "go",
    "module": "Qm123"
}   
```

Additionally, we can define several lens functions in a single module file.
```json
{
    "description": "Rename a field from source to destination",
    "url": "https://github.com/lens-vm/community-modules",
    
    "import": {
        "extract": "ipfs://Qm999"
    },
    
    "runtime": "wasm",
    "language": "go",
    "package": "ipfs://Qm123",
    
    "modules": [
        {
            "name": "rename",
            "arguments": {
                "type": "object",
                "properties": {
                    "source": {
                        "description": "The source field for renaming",
                        "type": "string"
                    },
                    "destination": {
                        "description": "The destination field for renaming",
                        "type": "string"
                    }
                }
            }
        },
        {
            "name": "rename",
            "arguments": {
                "type": "object",
                "properties": {
                    "source": {
                        "description": "The source field for renaming",
                        "type": "string"
                    },
                    "destination": {
                        "description": "The destination field for renaming",
                        "type": "string"
                    }
                }
            }
        },
    ]
}
```

## Lens File (lens.json/yaml)
A Lens File is the main entry point into the LensVM ecosystem. It defines all the lenses and their arguments to actually transform a document from one form to another. 

Example Lens File
```json
{
    "import": {
        "*": "Qm987",
        "hoist": "Qm654", // github.com/source/rename-go
        "convert": "Qm654"
    },
    "lenses": [
        {
            "rename": {
                "source": "body",
                "destination": "description"
            }
        },
        {
            "rename": {
                "source": "state",
                "destination": "status"
            }
        },
        {
            "convert": {
                "name": "status",
                "mapping": [
                    {
                        "open": "todo",
                        "closed": "done"
                    },
                    {
                        "todo": "open",
                        "doing": "open",
                        "done": "closed"
                    }
                ]
            }
        }
    ]
}
```

Lens files can be written in either `JSON` or `YAML`. Here we have `JSON` example with the two main fields `import` and `lenses`. 

### Importing Lenses
The `import` field defines all the lens modules we use in our Lens File. We can selectively import a named lens module defined in a Lens Definition, or import all the modules using the `"*"` label. Explicitly importing a named module always takes priorty over the same named module imported implicitly through a `"*"` label. Therefore, in the above example, the `hoist` module with ID `Qm654` would take precedent over an equivalently named `hoist` module implicity imported from the `"*"` label from the module `Qm987`.

If there are two implicity imported modules with the same name using a `"*"` label, priority is given to whichever has a the first lexigraphical ordering (E.g. `Qm111` < `Qm222`).

### Lens Parameters
The `lenses` field defines which lenses, the order, and the parameters to execute. These parameters must match the `arguments` field defined in the `Lens Module`, which is defined as a `JSON-Schema` to enable easy and consistent validation.

## Lens Graph

The Lens Graph is an emergent property in LensVM and not formally defined as a structure. Overtime, as Lens Files are defined and used in various systems a natural relationship from one lens file to another will exists. For Example, if LensVM is deployed in a Schema Registry environment, as more schemas are created, each with a lens file mapping from one to another, then indirectly the registry would contain a graph of schemas. This graph would contain lenses as edges and schemas as nodes. 

The `Lens Graph` would be a [Directed Acyclic Graph]() as each edge has an implied forward and reverse direction (based on the lens module)

![](https://www.inkandswitch.com/media/cambria/lens-graph.svg)
###### *image credit: https://www.inkandswitch.com/cambria.html*

In general, if a system uses LensVM, and is able to coorelate one lens to another through some kind of relation then LensVM can execute the entire traversal of related lenses as a single operation.

The LensVM project maintains a seperate toolkit called [lens-graph]() which is a utility to help developers construct and use Lens Graph's as defined above. 

## Application Binary Interface (ABI)
The ABI is a interface specification that ensures that compiled LensVM WASM modules seemlessly work with one another. It requires no changes to the existing WASM interface or specification, instead it states what functions are required to be exported via a WASM module.

At its core, LensVM modules are document transformation functions, which take in arguments defined in a `Lens File` and return a mutated document.

The LensVM ABI is split into `Module` functions and `Host` functions. `Module` functions are those that need to be implemented on the module side. `Host` functions need to be implemented on the host side.

## ABI - WASM Module Functions
A compiled LensVM WASM module must export the following functions

### `lensvm_module()`
- params:
    - none
- returns:
    - none

Indicates this a LensVM Module

### `lensvm_abi_version_X_Y_Z()`
- params:
    - none
- returns:
    - none

Exports ABI version.

`X, Y, Z` match `<major>, <minor>, and <patch>` cooresponds to the LensVM SemVer Version this module was compiled against (vX.Y.Z).

<!--
### `lensvm_exec`
- params: 
    - `context_id`: a unique identifier generated for each module execution
    - `lens_name`: the name of the lens module to execute
    - `arg_obj`: a CBOR serialized object containing the arguments passed in the `Lens File`.
    - `data_obj`: a CBOR serialized object containing the data to transform
- returns
    - `res_obj` is the return value, which is a CBOR serialized object with the results of the lens transformation
-->

Executes a named lens transformation, functionally equivalent to `lensvm_exec_<lens_name>`

### `lensvm_exec_<lens_name>`
- params: 
    - `context_id`: a unique identifier generated for each module execution
    - `arg_obj`: a JSON serialized object containing the arguments passed in the `Lens File`.
    - `data_obj`: a JSON serialized object containing the data to transform
- returns
    - `res_obj` is the return value, which is a CBOR serialized object with the results of the lens transformation

Executes a lens transformation where <lens_name> is the name of the lens, functionally equivalent to `lensvm_exec`. 

<!--
>[color=green] The reason for having an explicitly named lens export, and a general `exec` export is for module dependancies. If you define a module that depends on another module, we need to have a direct export for that function, otherwise we would have a name collision trying to import and execute multiple lenses, all of which used the `lensvm_exec` export name.
-->

### `lensvm_memory_return_ptr`
- params:
    - `return_ptr`: A return value from a function call, double pointer
- returns
    - `data_ptr`: A pointer to a data buffer in the WASM memory

Gets the pointer to a buffer from the return double pointer.

### `lensvm_memory_return_size`
- params:
    - `return_ptr`: A return value from a function call, double pointer
- returns
    - `data_size`: The size of the buffer pointer to by the return double pointer

Gets the size of the buffer from the return double pointer

## ABI - Host Module Functions
A compliant LensVM Host must export the following functions

### `lensvm_get_buffer`
- params:
    - `buffer_type`: The type of buffer to get
    - `offset`: The offset starting point in memory where the buffer exists
    - `max_size`: The max size of buffer
    - `return_buffer_data`: A pointer for the returned buffer
    - `return_buffer_size`: A pointer for the returned buffer size
- returns:
    - `call_result`: The status of the call

Host function to get a buffer value

### `lensvm_set_buffer`
- params:
    - `buffer_type`: The type of buffer to set
    - `offset`: The offset to set the buffer to in existing memory
    - `max_size`: The size of the existing buffer in memory
    - `buffer_data`: A pointer for the new buffer to set
    - `buffer_size`: The size of the buffer to set

Host function to set a buffer value

### `lensvm_exec_lens`
- params:
    - `name`: Name of the lens to execute
    - `forward`: Boolean indicating if this is a forward lens execution
    - `arg_obj`: a JSON serialized object containing the arguments passed in the `Lens File`.
    - `data_obj`: a JSON serialized object containing the data to transform
- returns:
    - `call_result`: The status of the call, which can be used to lookup the response value

Uses the host to call a defined lens module function. This is used for dynamic execution of lenses within a module, where it might not be possible to statically list the necessary imports in the Module file. However, any lens modules executed with this function must be defined in the Lens file imports.

## Memory Management
Memory is a complex issue in WebAssembly due to the isolation between host and module, and the nature of linear memory design of WebAssembly. This is compound by WASM only supporting very basic types (i32, i64, f32, and f64). Most notably WASM doesn't support complex types like arrays, objects, or even strings. 

Therefore, 

### Function Arguments
To use these complex types in our lens modules, we need to break them down into the base supported types, and utilize some memory management magic.

The WASM runtime supports sharing the linear memory system between the module and the host. The standard approach for supporting compound types is to serialize them into a byte slice/array, copying the buffer into the shared memory heap, and using supplying the new buffers pointer and size in function arguments

![](https://i.imgur.com/WGWiYTO.png)
###### *image credit: https://hacks.mozilla.org/2019/08/webassembly-interface-types/*

We then design our function as:
```go
// This function signature
someFunc(s string)

// Becomes
someFunc(sPtr *byte, size int)

// Which compiles to WASM
someFunc(i32, i32)
```

### Function Return

Return types are a little more complicated then arguments. The reason is that WebAssembly only supports returning a single value, unlike the arguments, which can contain several.

We can see from our approach from above, of converting our complex type into a pointer and a size argument needs to be adjusted to support a single value return system.

Additionally, we also need to support some kind of error/status return value. So, we need to include the following in a single return type
- Status or Error
- Pointer to string in memory
- Size of string in memory

The approach we take for LensVM is two fold. The single return value is a i32 type. We reserve the values 0-10 for error codes. If the return value is between 0-10, then we know there was some kind of error. If the value is > 10, then we know the execution was succesful.

For succesfull functional calls, the return value will be a double pointer (pointer to a pointer). The double pointer is an index into a map maintained by the module which points to both the pointer and size of the intended result type. We then have two helper functions `lensvm_mem_res_ptr` and `lensvm_mem_res_size`. 

#### `lensvm_memory_return_ptr(ret)`
A helper function which converts a return value double pointer to a concrete buffer pointer.

#### `lensvm_memory_return_size(ret)`
A helper function which converts a return value double pointer to a buffer size value

Finally, we use the host function`GetMemory(ptr, size)` to get the actual return value byte slice.

Here's an example to demonstrate how this all comes together
```go=
ret := module.someFuncThatReturnString()
if ret <= 10 {
    panic(StatusToError(ret))
}

ptr := lensvm_memory_return_ptr(ret)
size := lensvm_memory_return_size(ret)

buf := module.GetMemory(ptr, size)
val := string(buf)

```

### Managed Buffers
In addition to direct access memory for arguments and return values, LensVM host runtimes also manage a collection of buffers for use lens modules.

Managed buffers are the easiest method of data passing between host and module, as well as between modules. The main use of these buffers is to pass the `input` and `argument` data objects to the lens `exec` function.

Here is a breakdown of the available `Managed Buffers`, their intended use case, and their read/write permissions.


| Buffer Name | ID | Description | Permissions |
| -------- | -------- | -------- | -------- |
| InputData     | 0 | `JSON` object to be transformed by the lens     | read-only     |
| InputArg | 1 | `JSON` object containing the lens specific arguments from the lens.json file | read-only |
| OutputPatch | 2 | Output object containing the returned `JSON Merge Patch` produced by the lens | write-only |
| TempData | 3 | Temporary `JSON` object, wiped before lens function executions. Primarily used for inter-module lens execution. See [Temporary Buffers](#Temporary-Buffers) for more. | read-write |
| TempArg | 4 | Temporay `JSON` object, similar to `TempData` but used for argument data | read-write |


## Status Codes
As mentioned above, we can return an error from our function using a i32 return value, that also doubles as a pointer to return data. The values 1-10 are reserved for status codes, and everything is a pointer value. Additionally, the value 0 is reserved for `success` and no return value. The following table is a breakdown of the defined status codes.



| Code | Name | Description |
| -------- | -------- | -------- |
| 0     | `StatusOK`     | Succesful execution, and no return data     |
| 1     | `StatusErrInvalidContext`     | The given context id doesn't exist in the module     |
| 9     | `StatusErrUnknown` | An unknown error occured |

## Serialization
All serializtion of data between the WASM host and the LensVM module must be in JSON. We will revisit this requirement in the future as the WebAssembly spec evolves, and [interface-types]() proposal progresses.

JSON was chosen as it has wide support in every language. We will monitor the performance impact of constantly serializing to and from JSON to pass data.

