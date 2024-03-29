> ℹ️ Superseded by the [WebAssembly/stringref](https://github.com/WebAssembly/stringref) proposal.

# Universal Strings for WebAssembly

## Introduction

This document, provided for discussion, extends WebAssembly with a universal string type in order to:

* Achieve good / efficient interoperability between hosts, modules, JavaScript and Web APIs
* Achieve similar ecosystem benefits in WebAssembly as have languages running on the JVM or CLR, but for the entire WebAssembly ecosystem
* Avoid ecosystem fragmentation as would be introduced by separate mechanisms or imports on and off the Web
* Avoid alloc+copy->garbage at the boundary in between two Wasm GC-enabled languages and/or JavaScript (especially if a GC is not or cannot be highly tuned for this case)
* Avoid code size hits by having to handle strings explicitly at the boundary or shipping basic string functions and their dependencies with each module
* Avoid hurting developer experience, like having to author, publish, ship and/or install explicit adapters
* Avoid redundant re-encoding overhead when forwarding strings to multiple modules expecting different encodings (think npm)

It is one step towards universal modules that run the same everywhere, without having to recompile for different environments or having to maintain multiple standard libraries or abstractions.

## Motivation

Excellent interoperability with the Web Platform is critical for the long-term success of WebAssembly, and efficient interoperability between different programming languages running on various hosts is desirable. The most common higher data types used in between two modules, or a host like a browser and a module, are strings and arrays, so making sure that these perform well and are compatible with the bulk of languages is important. This document describes a mechanism for strings, addressing concerns raised in discussions like [[1](https://github.com/WebAssembly/interface-types/issues/13)] and [[2](https://github.com/WebAssembly/gc/issues/145)].

<p align="center">
  <img src="./module-graph.svg" alt="Module graph" />
</p>

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

* `string.is_well_formed <enc>` inquires whether a string is well-formed according to UTF-16.
  * `string.is_well_formed $e : [stringref] -> [i32]`

The list of instructions is not exhaustive and does not imply that instructions not yet mentioned aren't desirable. For example, instructions like `string.substring`, `string.concat` and others useful to perform common string operations may be desirable.

## Encodings

Encodings supported in the MVP of this document are:

Encoding | If well-formed | Immediate value | Encoding unit
---------|----------------|-----------------|---------------
[WTF-8](https://simonsapin.github.io/wtf-8/) | UTF-8 | 0x0 | 8 bits (u8)
[WTF-16](https://simonsapin.github.io/wtf-8/#wtf-16) | UTF-16 | 0x1 | 16 bits (u16)

The WTF family of encodings has been chosen over the respective UTF family of encodings because it is more lenient, i.e. does not introduce trapping behavior but defers sanitization to modules and APIs requiring it. JavaScript and most of its related APIs are effectively designed for WTF-16, not UTF-16, for example. Or as [The Unicode Standard, Version 13.0 – Core Specification](http://www.unicode.org/versions/Unicode13.0.0/ch02.pdf) states in section 2.7 Unicode Strings:

> Depending on the programming environment, a Unicode string may or may not be required to be in the corresponding Unicode encoding form. For example, strings in Java, C#, or ECMAScript are Unicode 16-bit strings, but are not necessarily well-formed UTF16 sequences. In normal processing, it can be far more efficient to allow such strings to contain code unit sequences that are not well-formed UTF-16—that is, isolated surrogates. Because strings are such a fundamental component of every program, checking for isolated surrogates in every operation that modifies strings can create significant overhead, especially because supplementary characters are extremely rare as a percentage of overall text in programs worldwide.

## Integration with linear memory based languages

The document does not impose the requirement of full GC support on a language using linear memory.

The `string.new` and `string.lower` instructions are useful at the boundary even if a module does not fully embrace or otherwise support GC, enabling interoperability with or between for example systems languages like C/C++ and Rust by legalizing the relevant instructions when

* Calling an imported function with a string argument using `string.new`
* Consuming a string argument in an export using `string.lower`

Furthermore, if there is a `string.new` creating a string from linear memory at one side of the boundary, and a `string.lower` immediately lowering the string at the other, as is the common case in systems languages, instead of creating an intermediate `stringref` the engine can optimize the operation to either

* A single copy from the source to the target memory if encodings match
* A re-encoding from the source to the target memory if encodings to not match

In addition, systems languages typically using linear memory may gracefully opt-in by providing a `wasm::string` backed by string instructions, may evaluate clever Reverse-Emscripten of for example `std::string` to `wasm::string` as a post-processing step, or, if neither is possible or desirable, rely on adapter fusion at the boundary to copy between memory and or GC objects for interoperability purposes.

## Implementation notes

Universal WebAssembly Strings as of this document can be implemented as a managed object with one slot per encoding. When a string from encoding A is created, only the slot of encoding A is populated. Accessing slot B will trigger re-encoding from A to B to populate slot B before using it.

* The common scenario is that each module uses exclusively encoding A or B, so populating the respective other slot typically happens at the boundary, but only has to be done once per string iff the other module is known to use a different encoding, or does not have to be re-encoded iff the other module is known to use the same encoding.
* The uncommon scenario is that there is a module using multiple encodings, i.e. conditionally A or B, in which case the engine must emit a branch when a string is accessed to populate the unpopulated slot before using it, or may pre-populate the slot at the boundary by speculating.
* Since the exact encoding required by a string instruction is known statically via its encoding immediate, the cost is either zero or a well-predicted branch triggering re-encoding once.
* Furthermore, an engine can omit superfluous checks and indirections if a preceeding string instruction in a code path already populates slot X, or if a `stringref`'s value is otherwise known to already have slot X populated.
* If a language or host requires well-formed strings (i.e. UTF-8 or UTF-16), it may either
  * Perform a check at the boundary and potentially sanitize a string. In the common case, the string is well-formed and does not require sanitization.
  * Deal with not well-formed strings within its standard library otherwise, like many programming languages and engines already do, which may be specific to the language's WebAssembly target
* In order to achieve ideal (speed and size) integration with linear memory based languages, an engine may split the code which makes a call with a string argument into a separate "adapter function" to avoid duplicating the entire function, and upon linking of two modules pick either the general or the optimized variant. Note that this particular mechanism is similar to what Interface Types proposes, but is implicit and as such improves developer experience and has zero code size cost.
* To avoid redundant work, an engine may cache a flag indicating that a string has already been checked for well-formedness. Checking can be performed implicitly during operations permitting it, like when re-encoding.

## Implementation alternatives

* Instead of maintaining multiple slots, a [Swift-like breadcrumb mechanism](https://swift.org/blog/utf8-string/#breadcrumbs) may abstract encoding differences away, using either only WTF-8 or WTF-16 for storage, potentially according to the host's native string encoding. This alternative has not been evaluated in detail yet. If feasible, it would need a way to represent lone surrogates similar to the WTF family of encodings.

---

[![Public domain](https://licensebuttons.net/p/zero/1.0/88x31.png)](http://creativecommons.org/publicdomain/zero/1.0/)

To the extent possible under law, [Daniel Wirtz](https://github.com/dcodeIO) has waived all copyright and related or neighboring rights to Universal Strings for WebAssembly.
