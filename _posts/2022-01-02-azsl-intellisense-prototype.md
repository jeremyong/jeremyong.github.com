---
layout: post
title: AZSL Intellisense Prototype
date: 2022-01-02 15:48
categories:
  - graphics
  - parsers
  - HLSL
  - AZSL
summary: Prototyping an intellisense framework for a shader language, for science
---

Just prior to the new year, I spent a couple days prototyping a quick-and-dirty implementation of Intellisense for AZSL.
As this prototype took me quite a bit afield of technologies I typically work with, I figured I'd document the process.
Plus, if it helps other graphics engineers write better tools in the ecosystem everyone benefits.

## Quick note about AZSL

In short, AZSL stands for the _Amazon shading language_ and is essentially HLSL with a few key extensions.
I originally intended to include more information about AZSL in this post, but have opted to leave that subject to a later time
to keep this post lean.
As the main subject of this post is the parsing/intellisense capabilities, I'll just briefly mention two key features which make AZSL different:
_Shader Resource Groups_ (i.e. _SRGs_) and _Shader Variants_.

SRGs allow shader bindings to be expressed in logical groupings that are composable through a new struct-like declaration (`ShaderResourceGroup`).
That is, if you have multiple shader fragments put together, they can each advertise a
set of resources they need (UAVs, SRVs, and samplers) and when the entire shader is compiled, a nice JSON file is produced with a
set of bindings needed to operate the shader in a platform agnostic manner. This is an abstraction on top of register/space semantics
in DirectX, or descriptor binding/set semantics in Vulkan.

The other key extension provided by AZSL are shader variants, expressed through a newly defined `option` keyword. Each shader
option exposed expands a "permutation-space" (more aptly named a "combination-space" in my opinion, but this isn't the term commonly used)
which amounts to a set of bits, one bit-per-option. At runtime, the "root shader variant" is used which allows the bitmask to be specified
at draw/dispatch time, and the branches are taken dynamically. However, each variant can then be specialized such that the dynamic branches
become static compile-time branches. This allows the programmer or artist to prototype rapidly, seeing results onscreen immediately by
leveraging the root variant, after which the static variants can be used for performance.

Note, I can't take any credit for either of these abstractions as they existed before I joined Amazon earlier this year.

## Writing a VSCode extension

What I wanted to achieve is shown below in screencast form. What's demonstrated is syntax highlighting for a number of the HLSL extensions
introduced by AZSL, include file navigation using F12, hover support for function declarations, and F12 navigate-to-definition functionality.

<video src="https://user-images.githubusercontent.com/250149/147891559-bc93acf9-5029-4d6d-911e-bc8304450724.mp4" data-canonical-src="https://user-images.githubusercontent.com/250149/147891559-bc93acf9-5029-4d6d-911e-bc8304450724.mp4" controls="controls" muted="muted" class="d-block rounded-bottom-2 width-fit" style="max-height:640px;max-width:100%" autoplay loop>
  </video>

At a high level, there are a few key pieces needed to put this 2-day prototype together:

- A Textmate grammar for basic syntax highlighting
- An LR(1) grammar for incremental parsing needed for semantic analysis
- A _Language Server Protocol_ (LSP) client, used to issue commands when the user performs various actions (e.g. editing a doc, hovering over a word, pressing F12, etc.)
- An LSP server which tracks document edits, parses, analyzes, and returns back useful information

Most of the code is written in [TypeScript](https://typescriptlang.com), which if you haven't heard of before, is pretty great (JS with static typing basically).

## The Textmate grammar

The full Textmate grammar for AZSL is [here](https://github.com/jeremyong/o3de-extension/blob/main/syntaxes/azsl.tmLanguage.json). Textmate grammars are
essentially a set of regex patterns, each of which runs on every line in the file to extract tokens and associate each token with a "name."
The name, in turn, is used by the editor to associate the token with a particular color. Here's an example of a very simple TM grammar:

```json
{
	"name": "AZSL",
	"scopeName": "source.azsl",
	"patterns": [
		{
			"name": "comment.line.double-slash.hlsl",
			"begin": "//",
			"end": "$"
		},
        {
			"name": "support.type.object.rw.hlsl",
			"match": "\\b(RWBuffer|RWByteAddressBuffer|RWStructuredBuffer|RWTexture1D|RWTexture1DArray|RWTexture2D|RWTexture2DArray|RWTexture3D)\\b"
		},
        {
			"name": "keyword.control.hlsl",
			"match": "\\b(break|case|continue|default|discard|do|else|for|if|return|switch|while)\\b"
		},
    ]
}
```

Obviously, this is nowhere close to the full grammar, given that it only supports a few patterns, but it should get the idea across.
At the top, we give our grammar a name (`AZSL`) and a scope name (`source.azsl`). The scope name allows us to reference this grammar
in other contexts in the future, should we choose to. For example, we might want to embed AZSL in a C++ or HTML file and highlight it
as such. Augmenting existing grammars to understand the `source.azsl` scope would allow us to do this, and is incidentally how syntax
highlighting works when you do things like embed code snippets in markdown files, as an example.

The first rule here is a simple begin/end pattern that starts with a `//` double-slash and continues to the end of the line (the `$`
regex token). This associates the pattern with the `comment.line.double-slash.hlsl` name. Because color themes all understand the
`comment.line` pattern, sequences of characters matching this begin/end sequence will be highlighted as such. We provide more specificity
than is necessary to disambiguate the pattern optionally, if we wanted a fancier color scheme for example.

The other rules in the short snippet above are regex based, matching against specific keywords that should be familiar. Obviously,
more complicated regexes are possible, but keep in mind that all these regexes are only permitted to operate _within a line_. Patterns
that span over multiple lines must use the `begin` and `end` style pattern.

For the AZSL TM grammar, I based it off the existing HLSL grammar [here](https://github.com/microsoft/vscode/blob/main/extensions/hlsl/syntaxes/hlsl.tmLanguage.json)
and made modifications needed for AZSL.

## Incremental parsing

To support any sort of intellisense-like capabilities, we need more than just regex-based pattern matching. With just regexes,
the matches have no understanding of semantic context. For example, just because I can match some parentheses, I have no way to know
if these parentheses are being used in a function declaration, a math expression, a call operation, a cast, or something else.
What's needed is a parser that generates an abstract syntax tree. Such a parser already exists as an [ANTLR](https://www.antlr.org/)
grammar as part of the AZSL compiler repo. You can see the ANTLR4 grammar for AZSL [here](https://github.com/o3de/o3de-azslc/blob/development/src/azslParser.g4)
and the AZSL token file [here](https://github.com/o3de/o3de-azslc/blob/development/src/azslLexer.g4).

What I wanted was a little different though. To be suitable for real-time intellisense, the parser needs to have a few key properties:

- Robust against parser failures (your code is in a non-compiling state most of the time as you edit)
- Fast (~1 ms for a full parse)
- Incremental (I don't want to parse the entire file as you edit)
- Bindings to the Node.js runtime

An amazing open source project which has all the above characteristics is [tree-sitter](https://tree-sitter.github.io/tree-sitter/).
TS was originally written for the Atom editor, but has since gained usage in a number of other editors and contexts. To use it though,
I needed to port the entire grammar to the TS grammar-DSL. After a few hours, I had [this](https://github.com/jeremyong/o3de-extension/blob/main/grammar.js).

If you've worked with parser generators before, the TS grammar-DSL will be quite familiar to you, and I found it very natural to
work with. After defining the grammar, you can generate native code via `npx tree-sitter generate --no-bindings`. The `--no-bindings`
flag prevents the CLI tool from generating Node.js bindings. I had to omit the auto-generated bindings because I discovered they were
not ABI compatible with the version of the Node runtime shipped with VSCode (╯°□°)╯︵ ┻━┻. I did manage to massage the bindings to
be compatible with the older headers/libs used by VSCode, but decided this was likely not worth the effort since it would mean I'd
have to bundle multiple binaries to ensure compatibility with whatever version of VSCode happened to be installed on the user's machine.

### WASM to the rescue

Another promising option was to cross-compile the generated parser code into [WebAssembly](https://webassembly.org/). This would have
a few benefits, chief of which is the ability to sidestep the ABI compabitility issue, while sacrificing just a bit of performance. As the
LSP architecture is fully asynchronous right off the bat, I wasn't as concerned with the performance loss, and after some testing and seeing
_sub-millisecond_ incremental parsing times, I decided this was "good enough." To cross-compile the code to WASM, the [https://emscripten.org/](Emscripten)
compiler is needed, and I found that I needed _specifically a 2.x version_ for the module to be loaded properly. Cross-compiling the
parser with the 3.x version resulted in a curious exception on load which I didn't have time to delve into. Luckily, TS even has WASM bindings
available [here](https://github.com/tree-sitter/tree-sitter/tree/master/lib/binding_web), so I didn't need to worry about bothering with the
mechanics of the WASM foreign function interface.

Below, I've ported a demo that parses the code live as you edit and displays the abstract syntax stree below. The embedded JS here fetches
the WASM module I created for parsing AZSL asynchronously, listens for changes to the text area, and parses the file in its entirety before
dumping the AST contents.

<script src="https://cdn.jsdelivr.net/npm/web-tree-sitter"></script>
<script>
const oldFetch = window.fetch;
window.fetch = function() {
    if (arguments[0].endsWith('tree-sitter.wasm')) {
        arguments[0] = 'https://cdn.jsdelivr.net/npm/web-tree-sitter/tree-sitter.wasm';
    }
    return oldFetch.apply(window, arguments);
}
async function main() {
    const Parser = window.TreeSitter;
    await Parser.init();
    const parser = new Parser;

    const wasmResponse = await fetch('https://raw.githubusercontent.com/jeremyong/o3de-extension/main/tree-sitter-azsl.wasm')
    const wasm = await wasmResponse.blob();
    console.log('WASM module loaded');

    const AZSL = await Parser.Language.load(new Uint8Array(await wasm.arrayBuffer()));
    parser.setLanguage(AZSL);

    const snippet = document.getElementById('snippet');
    const treeOutput = document.getElementById('tree');
    const parseTime = document.getElementById('parseTime');
    const parse = (text) => {
        const start = window.performance.now();
        const tree = parser.parse(text);
        treeOutput.textContent = tree.rootNode.toString();
        const end = window.performance.now();
        const time = end - start;

        parseTime.innerText = `Parsed in ${time} ms`;
    }
    snippet.addEventListener('input', (e) => {
        parse(e.target.value);
    });
    parse(snippet.textContent);
}
main();
</script>
<textarea id="snippet" style="width: 100%;" rows="30">
// EDIT ME
#include <Atom/Features/SrgSemantics.azsli>

ShaderResourceGroup InstanceSrg : SRG_PerObject
{
    row_major float4x4 m_objectMatrix;
    float4 m_color;
}

struct VSInput
{
    float3 m_position : POSITION;
};

struct VSOutput
{
    float4 m_position : SV_Position;
};

VSOutput MainVS(VSInput vsInput)
{
    VSOutput OUT;
    OUT.m_position = mul(InstanceSrg::m_objectMatrix, float4(vsInput.m_position, 1.0));
    return OUT;
}

struct PSOutput
{
    float4 m_color : SV_Target0;
};

PSOutput MainPS(VSOutput vsOutput)
{
    PSOutput OUT;
    OUT.m_color = InstanceSrg::m_color;
    return OUT;
}
</textarea>
<p id="parseTime"></p>
<p id="tree" style="font-family: monospace; height: 300px; scroll-y: auto; scroll-x: hidden; overflow-y: scroll;"></p>

Parsing incrementally relies on another piece to the puzzle, which is mirroring individual updates from VSCode to our
LSP server (which will run as a separate node process). One challenge here is that the edits are reported as line/character
tuples, but the text itself is consumed as a contiguous byte range. This means that we need some way of converting line/character
coordinates to offsets. While this could be accomplished efficiently using piece tables and prefix sums on line offsets, for this prototype,
I did the simpler conversion with string concatenation and line splitting. You can see this implementation [here](https://github.com/jeremyong/o3de-extension/blob/main/src/server/TextMirror.ts).

## LSP client/server architecture

The [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) is a nifty protocol that standardizes
how the text editor communicates user acitivity to a server process, which in turn returns back useful feedback needed to either
complete an action (e.g. jump to definition), colorize text semantically, provide hover tooltips, and more.

The extension I made advertises that it should activate whenever a file associated with AZSL is opened. To register the new file
associations, we can put this in the [`package.json`](https://github.com/jeremyong/o3de-extension/blob/main/package.json) file:

```json
    "languages": [
      {
        "id": "jsonc",
        "extensions": [
          ".pass",
          ".shader",
          ".shadervariantlist"
        ]
      },
      {
        "id": "azsl",
        "aliases": ["AZSL", "Azsl"],
        "extensions": [
          ".azsl",
          ".azsli",
          ".srgi"
        ]
      }
    ]
```

Note that there are several files we wish to interpret as AZSL files, and furthermore, we can even extend file associations for
existing known languages. Here, our `.pass`, `.shader`, and `.shadervariantlist` files are indicated as containing JSON-with-comments
data.

To activate our extension itself, we can specify activation events in the same `package.json` file:

```json
  "activationEvents": [
    "onLanguage:jsonc",
    "onLanguage:azsl"
  ]
```

Last, we need to actually spawn the LSP client and server itself. [Spawning the client](https://github.com/jeremyong/o3de-extension/blob/main/src/azsl.ts#L45) looks like:

```typescript
    let serverModule = context.asAbsolutePath(path.join('out', 'server', 'server.js'));
    let serverOpts: ServerOptions = {
        run: {module: serverModule, transport: TransportKind.ipc},
        // These options are used only when we are debugging the extension
        debug: {
            module: serverModule,
            transport: TransportKind.ipc,
            options: { execArgv: ['--nolazy', '--inspect=6009']}
        }
    };

    let clientOpts: LanguageClientOptions = {
        documentSelector: [{scheme: 'file', language: 'azsl'}],
        errorHandler: {
            error(error, message, count) {
                console.log(error, message);
                return ErrorAction.Shutdown;
            },
            closed() {
                console.log('Server closed');
                return CloseAction.DoNotRestart;
            }
        }
    };

    client = new LanguageClient('azslLanguageServer', 'AZSL Language Server', serverOpts, clientOpts);
```

The code above will spawn the server module we specify and use IPC as the transport mechanism.
If we wanted, we could host the language server remotely.

The server, after spawning, needs to create its end of the connection and initiate a handshake indicating what
capabilities it wishes to advertise. Here, we advertise the server as being a definition provider (F12 jump-to-definition)
and a hover provider (display a hover tooltip).

```typescript

let connection = createConnection(ProposedFeatures.all);

connection.onInitialize(async (params: InitializeParams) => {
    console.log('AZSL LSP client initializing');
    // Load our WASM module for parsing before proceeding
    await loadParser();

    const result: InitializeResult = {
        capabilities: {
            textDocumentSync: TextDocumentSyncKind.Incremental,
            definitionProvider: true,
            hoverProvider: true,
        }
    };
    return result;
});

// Add other callbacks here

connection.listen();
```

Note that the server indicates that it responds to incremental document syncs. This way, we avoid needing to pay
the cost of flushing the entire file on each edit through the IPC pipe.

Finally, we can implement the definition and hover providers as additional callbacks installed on the `connection`
instance. You can view the implementation of these callbacks [here](https://github.com/jeremyong/o3de-extension/blob/main/src/server/server.ts#L151).
Internally, they function by issuing queries to a cached representation of the document AST, which is updated
each time the document is edited. Shaders have include file support, so internally, we need to maintain a simple
symbol database that understands how to resolve include files to lookup symbols in other files. If I were to keep
on working on this extension, this is the area that needs the most improvement by far, as it's unlikely that my
simple approach of managing dictionaries and arrays will realistically scale (and it wouldn't surprise me if ther
are bugs).

## Conclusion

This was an "experiment" that has resulted in an alpha-quality public extension you can install in VSCode today (search for "O3DE"
in the extensions marketplace). As it's already been useful for me in navigating around shader files and headers, as well
as showing function signatures on hover, I've opted to publish it in this extremely incomplete state. Of course, given additional time,
there's plenty of functionality probably worth adding:

- Completion support
- Navigate to struct/class/enum declarations
- Displaying struct and struct member size/alignment information
- Diagnostics
- Faster text edit mirroring
- Symbol database backed by something like sqlite that could be persisted
- More robust include-file resolution

I would consider myself a novice in writing these sorts of tools, so take this post with a grain of salt and don't hesitate to reach out (social media links above)
if I've gotten anything wrong.
