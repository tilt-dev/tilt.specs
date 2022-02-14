# IDE Extensions
## Overview
An oft-requested feature is an improved `Tiltfile` editing experience in IDEs/editors.

Specifically, for the scope of this spec, we'll consider two high-level aspects:
 * Syntax highlighting
 * Advanced IDE features (autocomplete, go to definition, etc.)

Explicitly out of scope for consideration here is any kind of integration with `tilt up` like was done for the [vscode-tilt-status][] extension.
The focus here is on language-level (Starlark) static analysis.

## Targeted IDEs & Editors
Our primary target is Visual Studio Code due to its ubiquity and robust extension system.
Additionally, it's available as a target for embedding on the web, which makes it attractive to enable in-app Tiltfile editing in the future.
(Currently, this is not strictly a requirement or planned, but it has surfaced in product conversations multiple times and a POC was built in summer 2021.)

When making technical decisions, however, it's important to consider the broader IDE ecosystem to enable additional extensions for other IDEs & editors while avoiding creating an unsustainable maintenance burden.
More concretely, we should avoid writing as much vendor-specific code as possible.

The secondary targets the JetBrains family of IDEs (IntelliJ, GoLand, PyCharm, etc).
Note that these are referred to uniformly under the umbrella of "IntelliJ" throughout the remainder of this document, as this is the underlying base for all JetBrains IDEs.

All other targets, including popular terminal-based editors like vim or Emacs are excluded for the moment.

However, whenever possible, we should strive to make choices enabling broad editor support in the future without bikeshedding or falling for YAGNI. 

## Syntax Highlighting
The [TextMate Grammar][textmate-grammar] format is popularly used as an interchange format for language syntax highlighting.

There is a robust, official TextMate Grammar for Starlark available in the [`syntaxes/` subdirectory of the `vscode-bazel` extension][vscode-bazel-syntaxes].

VSCode uses this natively as outlined in the [VSCode Extension Syntax Highlighting Guide][vscode-syntax-highlighting].

IntelliJ has native support for loading TextMate bundles.
_However,_ this is not the mechanism extensions are expected to use; JetBrains has their own codegen tool, [Grammar-Kit][grammar-kit], to convert a BNF to a compatible parser.

### Conclusion
Given the existence of an official TextMate grammar combined with its necessity to support VSCode, this is sufficient for now.
As we won't have a native IntelliJ extension right now, providing instructions for IntelliJ users to load the TextMate bundle is sufficient.
In the future, if we develop an IntelliJ extension, we can investigate programmatically registering TextMate bundles or alternative approaches.

## Advanced IDE Features
Historically, providing advanced IDE features has meant a lot of code duplication and maintenance, as there was no standard and IDEs are written in a variety of languages.

Currently, there are no Starlark (or derivatives, e.g. Bazel) open-source IDE extensions using either LSP or vendor-specific techniques.

In recent years, Microsoft has championed the [Language Server Protocol][lsp-docs], which is attractive for several reasons:
 * Language analysis logic can be written in any language
 * Async/non-blocking API (prevent editor freezes)
 * Reduces vendor-specific extension code
 * Capability based (do not have to support all possible features)

(These are not the only benefits, e.g. there are some nice security/stability side effects from running the LSP server as a separate process.)

VSCode has leaned heavily into using LSP for language extensions.
As a result, the VSCode LSP client handles _everything_ already - no bridge code between LSP<>VSCode is required.

This makes LSP extremely attractive for our use case.
First, it opens the possibility of re-use of the LSP server for future IDE extensions.
Additionally, and perhaps more importantly, it dramatically simplifies development of the VSCode extension in particular, as [vscode-languageserver-node][] handles everything for the editor UX and is battle-tested.

IntelliJ does not have robust _first-party_ LSP support.
The [lsp4intellij][] project provides a bridge between LSP<>IntelliJ for extensions (similar to [vscode-languageserver-node][]).
Its maintenance/development status is questionable as of February 2022.
Additionally, there is a generic [intellij-lsp][] plugin, so a motivated user could configure it manually to use our LSP server now.
Its maintenance/development status is similarly questionable as of February 2022.

### Conclusion
While editor support beyond VSCode is still immature, LSP is quickly growing in popularity, especially as it makes it practical to support "niche" languages such as Tiltfile-Starlark.

Even in a VSCode-only world, LSP is a desirable target because of the abstractions provided.
This will reduce the amount of error-prone UI interfacing code we need to write and dramatically simplify testing.

The remainder of this document will explore nuances of developing an LSP server.

## Language Server Protocol (LSP) Deep Dive
LSP is a relatively recent standard (the first public release was in 2017) for a complex topic (programming language analysis).
As a result, the ecosystem is a bit chaotic: there are often multiple competing LSP servers for a given language and high-level documentation can be sparse.

Luckily, the [LSP specification][lsp-spec] itself is extremely readable, and the JSON-RPC2 protocol makes introspection human-friendly.

### Go Support
The [go.lsp.dev][] project provides a JSON-RPC2 server and client implementation in addition to Go structs for the LSP wire messages.
It's an exported version of the inaccessible implementation used for the official Go LSP server (`gopls`) from [golang.org/x/tools/internal/lsp][x-tools-lsp].
Note: it appears to have diverged at this point; stewardship and affiliation (if any) to Go team is unclear as of February 2022.

Developing an LSP in Go is a natural choice:
 * Possible to integrate directly into Tilt (`tilt lsp` command) eliminating extra dependencies
 * Tilt team familiarity with Go
 * Official Starlark parser implementation exists
   * See caveats in [Starlark Parsing](#starlark-parsing) section below

### Starlark Parsing
While writing code, it's very normal for the file to be syntactically invalid.
This is often at odds with language parsers, which strictly follow a formal grammar.

Microsoft wrote a fantastic [overview of lessons learned][fault-tolerant-parser] from writing a fault tolerant PHP parser for use in VSCode.

#### starlark-go
The [starlark-go][] parser stops at the first error it encounters.
Furthermore, it does this via calling `recover()` from a `panic()` rather than more idiomatic Go error-handling.

That said, the starlark-go parsing code is hand-written and roughly ~1000 LOC, so creating a fault-tolerant version is not insurmountable.
It's unlikely we'd be able to upstream this, but the Starlark language spec is a slow-moving target, so the ongoing maintenance burden here would be minimal.

#### Tree-sitter
Another approach worth exploring is to generate a [Tree-sitter][] grammar.
Tree-sitter is designed for fast parsing, incremental updates, and graceful error recovery (all important properties for an LSP server).
It's written in dependency-free C, which makes bindings for a variety of languages practical including Go and WASM.

Microsoft even uses it in the [vscode-anycode][] extension, which provides generic LSP features using _any_ Tree-sitter AST.

Tree-sitter also has highlighting functionality that can be used to provide semantic syntax highlighting that's more accurate than the TextMate-based regex approach.
(This would also be useful for a future IntelliJ extension: see the notes in the [Syntax Highlighting](#syntax-highlighting) section.)

### Conclusion
We should write our LSP in Go.

On the LSP protocol & communication: the LSP Go packages, while under-documented, are feature complete and real world tested via `gopls`.
(Note: this is specifically in reference to the protocol & JSON-RPC2 communication aspects, not the actual language analysis logic in `gopls`.)

On Starlark analysis: there are no viable Starlark parsers written in another language: both the Go and Java implementations have extremely strict, hand-rolled parsers, and the Rust implementation is now maintained under the `facebookexperimental` org and has some syntax differences.
The most compelling, modern generic parser/AST library, Tree-sitter, is written in C, and has bindings to many languages, including Go.

We can use the existing Python Tree-sitter grammar to prototype `Tiltfile` LSP functionality (e.g. autocomplete).
If successful, we can adapt the Python Tree-sitter grammar to the Starlark dialect.
If unsuccessful, we can experiment with forking the `starlark-go` parser and making it more lenient. 

## Other Approaches
### VSCode Python Extension
A Tilt user showed off their VSCode setup in ([tilt-dev/tilt#4734][github-tilt-4734]):
 * VSCode Python extension
 * Tilt API definitions ([`api.py`][tilt-api.py]) converted to Python type stubs (`.pyi`)
 * Magic `import` statement in `Tiltfile` to trigger code completion
   * Not valid Starlark; must be removed before Tiltfile is executed

This configuration as-is has two big downsides:
 1. Potentially incorrect/misleading syntax error reporting where Starlark and Python differ
 2. Necessity of magic `import` statement that must be manually added and then removed while editing

If we were to adopt this as a more general Tilt-sanctioned approach, it would also necessitate users having a functional Python installation, as the underlying analysis/parser code is written in Python.
Beyond being an extra requirement, the end-user Python packaging and distribution ecosystem is fraught with issues, which will inevitably increase the support burden.

Furthermore, while it's likely a Tilt extension taking this approach could obviate the need for a magic `import` statement, it's unlikely we'd be able to adequately reconcile semantic/syntactical differences between Python and Starlark.
For example, Python uses `import` while Starlark has `load` (in addition to the Tilt-specific `load_dynamic` and `include` functions).
To handle cross-file symbol resolution properly, the underlying Python tooling would presumably need to be forked (and then maintained).

[github-tilt-4734]: https://github.com/tilt-dev/tilt/issues/4734
[fault-tolerant-parser]: https://github.com/microsoft/tolerant-php-parser/blob/f4f5e9303253b7e60a52e8078cba658a23c042e8/docs/HowItWorks.md
[go.lsp.dev]: https://go.lsp.dev
[grammar-kit]: https://github.com/JetBrains/Grammar-Kit
[intellij-lsp]: https://github.com/gtache/intellij-lsp
[lsp-docs]: https://microsoft.github.io/language-server-protocol/
[lsp-spec]: https://microsoft.github.io/language-server-protocol/specifications/specification-current/ 
[starlark-go]: https://github.com/google/starlark-go
[textmate-grammar]: https://macromates.com/manual/en/language_grammars
[tilt-api.py]: https://github.com/tilt-dev/tilt.build/blob/3670b569adad8553ad990358c0ef6e7aa0fda56a/api/api.py
[Tree-sitter]: https://tree-sitter.github.io/tree-sitter/
[vscode-anycode]: https://github.com/microsoft/vscode-anycode
[vscode-bazel-syntaxes]: https://github.com/bazelbuild/vscode-bazel/tree/master/syntaxes
[vscode-languageserver-node]: https://github.com/Microsoft/vscode-languageserver-node
[vscode-syntax-highlighting]: https://code.visualstudio.com/api/language-extensions/syntax-highlight-guide
[vscode-tilt-status]: https://github.com/tilt-dev/vscode-tilt-status
[x-tools-lsp]: https://pkg.go.dev/golang.org/x/tools/internal/lsp
