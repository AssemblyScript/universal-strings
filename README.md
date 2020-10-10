# Definitely not a Universal String Proposal for WebAssembly

## Introduction

This document extends WebAssembly with a universal string type in order to achieve:

* Good / efficient interoperability between hosts, modules, JavaScript and Web APIs
* Avoid alloc+copy->garbage at the boundary in between two Wasm GC-enabled languages and/or JavaScript
* Achieve similar ecosystem benefits in WebAssembly as have languages running on the JVM or CLR

It is one step towards universal modules that run the same everywhere, without having to recompile for different environments or having to maintain multiple standard libraries or abstractions.

## Motivation

Good interoperability with the Web Platform is critical for the long-term success of WebAssembly, and good interoperability between different programming languages running on different hosts is desirable. The most common higher data types used in between two modules, or a host and a module, are strings and arrays, so making sure that these perform well and are compatible with the bulk of languages is important. This document describes a mechanism for strings, addressing concerns raised in discussions like [[1](https://github.com/WebAssembly/interface-types/issues/13)] and [[2](https://github.com/WebAssembly/gc/issues/145)].

## Types

* `string` is a new heap type
  * `string <: data`

* `stringref` is a new reference type
  * `stringref == (ref string)`
  * `(ref null string)` is the respective nullable variant

## Instructions

* `string.new <enc>` allocates a new string from units of the given encoding, stored at the specified region in memory.
  * `string.new $e : [i32, i32] -> [stringref]`

* `string.get <enc>` obtains the value of the encoding unit at the specified offset of a string.
  * `string.get $e : [stringref, i32] -> [i32]`

* `string.len <enc>` inquires the length of a string in encoding units in respect to the specified encoding.
  * `string.len $e : [stringref] -> [i32]`

* `string.lower <enc>` lowers a string's encoding units to the specified offset in memory.
  * `string.lower $e : [stringref, i32] -> []`

* `string.eq <enc>` compares two strings for equality given the specified encoding.
  * `string.eq $e : [stringref, stringref] -> [i32]`

The list of instructions is not exhaustive and does not imply that instructions not yet mentioned aren't desirable.

## Encodings

Encodings supported in the MVP of this document are:

Encoding | Immediate value | Encoding unit
---------|-----------------|---------------
[WTF-8](https://simonsapin.github.io/wtf-8/) | 0x0 | u8 / 8 bits
[WTF-16](https://simonsapin.github.io/wtf-8/#wtf-16) | 0x1 | u16 / 16 bits

The WTF family of encodings has been chosen over the respective UTF family of encodings because it is more lenient, i.e. does not introduce trapping behavior but defers sanitization to modules and APIs requiring it. JavaScript and most of its related APIs are effectively designed for WTF-16, not UTF-16, for example.

## Implementation notes

Universal WebAssembly Strings as of this document can be implemented as a managed object with one slot per encoding. When a string from encoding A is created, only the slot of encoding A is populated. Accessing slot B will trigger re-encoding from A to B to populate slot B before using it.

* The common scenario is that each module uses exclusively encoding A or B, so populating the respective other slot typically happens at the boundary, but only has to be done once per string iff the other module is known to use a different encoding, or does not have to be re-encoded iff the other module is known to use the same encoding.
* The uncommon scenario is that there is a module using multiple encodings, i.e. conditionally A or B, in which case the engine must emit a branch when a string is accessed to populate the unpopulated slot before using it, or may pre-populate the slot on the boundary by speculating.
* Since the exact encoding required by a string instruction is known statically via its encoding immediate, the cost is either zero or a well-predicted branch triggering re-encoding once. Furthermore, an engine can omit superfluous checks if a preceeding string instruction in a code path already populates slot X.
* If a language or host requires well-formed strings (i.e. UTF-8 or UTF-16), it may either
  * Perform a check at the boundary and potentially sanitize a string. In the common case, the string is well-formed and does not require sanitization.
  * Deal with not well-formed strings within its standard library otherwise, like many programming languages and engines already do, which may be specific to the language's WebAssembly target
