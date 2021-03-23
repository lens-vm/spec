<p align="center">
<img src="https://i.imgur.com/u29eC7t.png">
</p>

#
LensVM is a data migration/evolution engine powered by bi-directional lenses. This work is based on the [Cambria]() project by Ink&Switch. LensVS uses bi-directional lenses, written in WebAssembly for portability to transform data from one shape to another. This project houses a collection of specifications and implementations to realize the full goal of LensVM. This project relies on [Content Identified]() data models, [JSON-Schema](), and (maybe) [Ceramic Network]() for data consistency and schema registration.

## Motivation
When dealing with schemas and data shapes from multiple different sources, applications, or versions; centralized or decentralized, can become a nightmare of interopability as schemas quickly diverge from one another. This is seen in database migrations, application version evolution, and platform interactions, where time and time again two systems become completely independant from one another with little option for reconciliation without data loss on one or both sides.

## Goals
The goal for LensVM is to create a consistent system, that can operate in both centralized and decentralized environments, which allows developers to easily transform data from one structure to another, without mass coordination, and using a standards-first approach. 

The first implementation will allow bi-directional lenses to be compiled to WebAssembly from several origin languages like C++, Rust, and Go, and execute these lenses in any host environment that supports the WASI interface to succesfully transform data defined using JSON-Schema from one schema to another.

## Bi-Directional Lenses
Bi-Directional lenses that power LensVM are based on the work by the [Cambria]() project from [Ink & Switch] industrial research lab. You can read their essay on their implementaion and goals [here](). To summarize, these lenses are succinct transformations which can be applied in a forward and reverse direction to modify JSON objects field by field from one schema to another. This includes renaming fields, moving fields from sub objects, mapping fields from one type to another (e.g. scalar to array), and more.

These lenses operate in an isolated layer, with minimal additional meta-data to reduce complexity and dependancy on other elements of the software stack. 

Each lenses executes a single transformation, which can then be composed into more complex lenses, or into a array of lenses to be executed.

![](https://i.imgur.com/wbhKoLx.png)

## Composability & Execution
The main purpose of LensVM is to freely allow lenses to be written in one language, and used in any other language via WASM. Moreover, lens design is conducted in an open environment by developers in a bottom up grass roots community, akin to Ethereum EIP proposals. 

Lenses are accompanied by their matchin meta-data which defines their functional goal, parameter types, and dependancies on other lenses. All compiled lenses and meta-data are stored in a Content Identified data model like [IPLD](). This allows developers to pin-point a specific instance or version of a lens, and never have that dependancy broken by future updates.

LensVM uses language specific SDKs to create lens functions to ensure consistency and utility between languages, and host VM's. This SDK will provide hooks to import lenses from other langauges via the Lens Definitiona and compiled WASM code. It will provide consistent utility functions to manage incoming data and parameters, and translate that to a Lens File.

## Lens Definition
A Lens Definition is a combination of the compiled lens in WASM, and its associated meta-data into a single CID based object. A lens definition only exports a single transformation (like `rename`), however that exported function can itself depend on other pre-existing lenses, identified by the CIDs of their respective Lens Definition files. These definitions can then be importent individually or as a group defined as a Lens Package.

Before formally defining a `Lens Definition` here's a rough example

```json
{
    "name": "rename",
    "description": "Rename a field from source to destination",
    "arguments": {
        "source": {
            "description": "The source field for renaming",
            "type": "string"
        },
        "destination": {
            "description": "The destination field for renaming",
            "type": "string"
        }
    },
    
    "runtime": "wasm",
    "origin": "go",
    "module": " ... " // CID Link or raw bytes?
}   
```