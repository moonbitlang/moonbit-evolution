# moon.pkg DSL
* Proposal: [ME-0012]() 
* Author: [Yorkin](https://github.com/Yoorkin)
* Status: Draft
* Review and discussion: [Github](https://github.com/moonbitlang/moonbit-evolution/issues/18)

## Introduction

### Module and Package

In a MoonBit project, a module is a collection of packages, and a package is the minimum compilation unit. Each package has a namespace. The Moon build system provides two configuration files:

* `moon.mod.json`

  A module descriptor file that contains the dependencies and metadata of the whole project.

* `moon.pkg.json`

  A package descriptor file used to import packages (including regular import, blackbox test import, and whitebox test import), specify link flags, virtual packages, and so on.

### Characteristics of package descriptor

The current design of `moon.pkg.json` has two key characteristics, which will not be discussed and will remain unchanged in this proposal:

* **Build system speed optimization**
    
    The import declarations are written in `moon.pkg.json` , rather than being spread across the source files. This design reduces the moon build system's IO operations during dependency graph generation and ensures the project can be built quickly.

* **Package scope semantics**
    
    In some programming languages, import declarations are file-scoped, requiring users to repeat imports across source files within the same package. In MoonBit, imports are package-scoped, eliminating this repetition.

## Motivation

Since imports are used frequently, users have to edit `moon.pkg.json` during development. The current JSON format can be verbose and annoying to write:

```json
{
    "import": [
        "path/to/pkg1",
        "path/to/pkg2",
        { "path": "path/to/pkg3", "alias": "alias1" },
        { "path": "path/to/pkg4", "alias": "alias2" }
    ],
    "test-import": [
        "path/to/pkg5",
        { "path": "path/to/pkg6", "alias": "alias3" }
    ],
    "wbtest-import": [
        "path/to/pkg7",
        { "path": "path/to/pkg8", "alias": "alias4" }
    ],
    "is-main": true,
    "alert-list": "+a+b-c",
    "bin-name": "my_binary",
    "bin-target": "target",
    "implement": "virtual_pkg_name",
    "supported_targets": ["target1", "target2"],
    "targets": {},
    "link": {},
    "native_stub": ["a", "b" ,"c"]
}
```

## Proposed solution

We propose introducing a DSL called `moon.pkg` to replace the current `moon.pkg.json` format. The new DSL allows comments, trailing commas, and provides shorthand syntax for frequently used, stable configurations, such as import declarations. Below is an example of the proposed `moon.pkg` syntax:

```moonbit
import {
  "path/to/pkg1",
  "path/to/pkg2",
  "path/to/pkg3" as @alias1,
  "path/to/pkg4" as @alias2,
}

import "test" {
  "path/to/pkg5",
  "path/to/pkg6" as @alias3,    
}

import "wbtest" {
  "path/to/pkg7",
  "path/to/pkg8" as @alias4,
}

config {
  "is_main": true, // allow comments
  "alert_list": "+a+b-c",
  "bin_name": "name",
  "bin_target": "target",
  "implement": "virtual-package-name",
  "supported_targets": ["wasm", "js"],
  "targets": {},
  "link": {},
  "native_stub": [
    "a", "b", "c"
  ], // allow trailing commas
}
```

### The import declarations

The `import` declarations is simple. The most common usage is to import packages without aliasing:

```moonbit
import { 
  "moonbitlang/async/io",
}
```

To import with aliasing, we use the `as` keyword followed by `@alias` .
The alias must start with `@` becuase the package names may not be a valid identifier, like `test` .

```moonbit
import { 
  "path/to/pkg/test" as @test,
}
```

To declare import for blackbox tests and whitebox tests, we use the `import "test"` and `import "wbtest"` blocks.

### The config block

Since configuration options are not yet stable and may change in the future, we cannot design special syntax for each option at this time. Instead, we use a `config` block that contains key-value pairs similar to JSON syntax, which forms a subset of the MoonBit language.

This design maintains compatibility with current options and allows easy extension of experimental configuration options in the future without requiring syntax changes.

Once the build system and configuration become stable, we can consider providing special syntax for frequently used options to improve usability.

### Benefits 

* **Better user experience for import declarations**

    The syntax is easier to write and supports comments and trailing commas.

    It can be integrated as part of MoonBit syntax. In a regular MoonBit project, 
    if `import` appears at the top of a source file, the parser can recognize 
    it and provide a code action to automatically move the import to `moon.pkg` . 

* **100% compatible with current options**

* **No more general-purpose configuration language**
    
    The `moon.pkg` DSL is designed specifically for MoonBit. The syntax is a subset of the MoonBit language (except the `config` keyword), making it intuitive for users familiar with MoonBit. It also avoids the complexity and overhead of general-purpose configuration languages like YAML or TOML.

* **Enables future script mode support**
     
    In the future, we may support single script file mode for MoonBit. In that mode, `import` declarations can be used directly in source files, eliminating the need to create a separate `import` syntax.

### Disavantages

The DSL makes external tools harder to parse the `moon.pkg` configuration. This can be mitigated by providing a command in the build system that parses `moon.pkg` files and outputs their JSON representation.

## Conclusion

The `moon.pkg` DSL addresses the practical pain points of working with package imports by replacing the verbose JSON format with a more intuitive syntax, and enables script mode support in the future. It maintains backward compatibility and the performance characteristics of the current build system, making package management more developer-friendly without compromising MoonBit's core design principles.
