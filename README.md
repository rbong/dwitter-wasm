# dwitter-wasm

This script turns [WebAssembly](https://webassembly.org/) binary files into a format consumed by [Dwitter](https://www.dwitter.net/) in as short a form as possible.
This should allow for fast performing Dweets, assuming you can fit something into the binary code and do anything with it in the remaining character limit.

## System Requirements

- [Node.js](https://nodejs.org/) 4.0.0+
- [Bash](https://www.gnu.org/software/bash/)

## Recommended Tools and Reading

- [WasmFiddle](https://wasdk.github.io/WasmFiddle/) for interactively building `.wasm` files for experimentation
- [WebAssembly Binary Toolkit](https://github.com/WebAssembly/wabt) for compiling `.wat` files locally and additional operations
- [The WebAssembly docs](https://webassembly.org/docs/js/)
- [MDN WebAssembly reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly)

## Dweet Limitations

The current version uses a minimum of 103 Dweet characters out of 140.
Characters are compressed down to Unicode characters, which allows for double the characters in Dweets.
This leaves you with 74 Javascript characters and WebAssembly binary characters.

To get the character count as low as possible, browser compatibility has been sacrificed.
The use of WebAssembly also comes with browser limitations.
Please note the compatibility on
[TypedArray.from()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/from#Browser_compatibility),
[WebAssembly.Module](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module#Browser_compatibility),
and [WebAssembly.Instance](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Instance#Browser_compatibility)
in order to determine which browsers your Dweet will work in.
At the time of writing, you can expect the Dweet to work best on the desktop in Firefox and Chrome.

## Installation

```bash
$ npm install --global dwitter-wasm
```

## Usage

**Basics**

To begin, create a `.wasm` source file using your tools of choice.
This binary file will be used to create a [WebAssembly.Instance](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Instance) in the final output.

You must then create a `.js` source file, which should contain any additional Javascript code for the Dweet.
The `WebAssembly.Instance` `exports` created out of the provided `.wasm` file will be available through the Javascript variable `m`.
The Javascript code cannot currently contain Unicode characters.

By default, the Javascript source file will be appended onto the end of the generated code which loads the WebAssembly binary separated by a semicolon.
However, if you include the string \`WASM_INPUT\` in the Javascript source file, the encoded WebAssembly binary will be injected instead.
This method of combining the source files can be used if you would like to pass imports to the WebAssembly module or make other customizations to the WebAssembly binary loading code.
An example template with the loader code size optimizations can be generated with `dwitter-wasm --template`.

You must also specify an output path for the location where the final Dweet code will be contained.
Since the Dweet uses specially encoded characters, you must copy the code into Dwitter using a program that supports Unicode.

Example using `wat2wasm` and [xclip](https://github.com/astrand/xclip):

```bash
$ echo "(module)" >> input.wat
$ wat2wasm input.wat -o input.wasm
$ echo "console.log(m)" >> input.js
$ dwitter-wasm -w input.wasm -j input.js -o output.js
$ xclip output.js -sel clip
```

Using this example, the Dwitter code will be copied to your clipboard for pasting to Dwitter.
The Dweet will do nothing but print `m` to your console.

**Command Line Arguments**

- `-w|--wasm-input`: The `.wasm` file to encode and embed in Javascript code
- `-j|--js-input`: The `.js` file to append to the generated Javascript code
- `--encode-wasm`: Only perform the step where the `.wasm` is encoded and embedded and combined with Javascript.
                   By default, all steps are all enabled.
- `--compress`: Only perform the step where the code is compressed, using `--js-input`.
                By default, all steps are all enabled.
- `-o|--output`: The resulting `.js` file
- `--template`: Output an optimized template for input and exit
- `--version`: Output the version number and exit
- `--help`: Output a help string and exit

## How it Works

**The WebAssembly.Instance Initialization**

First, the WebAssembly binary is turned into a valid Javascript string using special ASCII characters.
Then, the string is turned into a [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array).
This format is required.
The array is then passed into a [WebAssembly.Module](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module).
The module is then passed into a [WebAssembly.Instance](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Instance).
The exports are then retrieved from the instance and stored in variable `m`.

For the most part, this part of the process is similar to [regular synchronous initialization of WebAssembly binaries](https://medium.com/@mbebenita/hello-world-in-webassembly-83951757775).
However, the `Uint8Array` is initialized from a string using special ASCII characters with keycodes corresponding to WebAssembly binary values.
This saves on characters by eliminating the need to define the binary values in comma-separated format.

**Code Compression**

After the WebAssembly binary initialization code is generated, it is concatenated with the Javascript source code.
At this point, the code is changed from single-width ASCII characters into double-width Unicode characters.
Since a single Unicode character can contain two ASCII characters, and Dwitter counts Unicode characters as single characters, this effectively halves the code.
