# jspreproc API

This guide includes the property name of the options object passed to the jspreproc function, the type(s) of the value, their default value, and a short example.

All examples assume jspreproc is instanced with this code:

```js
var jspp = require('jspreproc')
```


## The main jspreproc function

Basic syntax:

```js
stream = jspp(string | array | stream [, object])
```

First parameter is the input, the second one are the options object. The output of `jspp` is a stream.

Each call to `jspp` maintains its own options and variables, independent of other calls, and a single input (the _mainstream_), whatever the number of files to be processed.

### Input

The input parameter can be one of this types:

* **string** - The name of a file in the file system to be readed by jspp. The file content is readed in utf8 mode with the [`fs.readFile`](https://nodejs.org/api/fs.html#fs_fs_readfile_filename_options_callback) function of node.
* **array** - This is an array of file names. These names become #include directives concatenated into an internal buffer, used by jspp as its mainstream.
* **stream** - Any readable stream, jspreproc set its encoding to utf8. As with a file, the stream is read completely before starting the processing.    

Default extension for file names is `.js`. Path can be absolute or relative to the file being processed. The current directory is used to resolve relative paths of included files in the mainstream only, as long as this is a stream.

_Example:_
```js
// Current directory is /apps/myapp (or C:\apps\myapp for Windows)
// The input is a stream, contains:
//#include file1              // => /apps/myapp/file1.js
//#include ..\shared.txt      // => /apps/shared.txt
```
file1.js:
```js
//#include file2              // => /apps/myapp/file2.js
//#include fixes/file3        // => /apps/myapp/fixes/file1.js
```
file3.js:
```js
//#include more/file4         // => /apps/myapp/fixes/more/file1.js
```
...and so on.

There's another way to set the jspreproc mainstream: as a string through the buffer property of the options, but this is most useful for testing and debugging.

If neither of these values is given (you can send `null` or the empty string as parameter), jspp will use the standard input.


### Options

The second parameter, the options, is an object containing configuration data and variables for the operation of jspp over _the current_ mainstream. Each call to `jspp()` resets its values, including defined jspreproc variables.

Detail for the properties is in the next part of this document.


### The Result

The returned stream is a generic [`stream.PassThrough`](https://nodejs.org/api/stream.html#stream_class_stream_passthrough) instance, with default encoding to utf8. You can redirect it to wherever you want.

_Example_:

```js
var jspp = require("jspreproc");
var fs = require("fs");
var st = fs.createWriteStream("dist/app.js");

jspp(["lib/file1.js", "lib/file2.js"], {set: "RELEASE", emptyLines: 0}).pipe(st)

```

When an error is generated by an incorrect parameter, an invalid variable name, or an error evaluating an expression, the error is emitted through the node standard mechanism for streams, so you have to define a handler for the "error" event to catch these.


---

## Properties (of options)

This is the list of properties for the `options` object passed as second parameter to jspp.

**NOTE:** `#define` is deprecated

Due to the very different behavior of the `#define` directive of jspreproc and the C preprocessor, this keyword is replaced by `#set` and, for consistency, `#undef` is replaced with `#unset`.

`#define` will be recognized in all 0.2.x versions, and starting from 0.3.0 will be removed. Perhaps in the future it is implemented with a similar behavior to the C preprocessor.


### varset

Creates or modifies jspreproc variables whose value can be used in conditional expressions or for replacement on the source.

type          | default | example
--------------|---------|---------
object, see bellow for value types | -- | `{varset {MODULE: "one"}}`

Valid names begins with one dollar sign, underscore, or ASCII _uppercase_ alphabetic character (`$`, `_`, `A-Z`), followed by one or more underscores or uppercase alphanumeric characters (`_`, `A-Z`, `0-9`).

If the name begins with `$_`, followed by one or more valid characters, you can use the name for replacing text in the source, too. The replacement is performed with the literal value of the variable at the time of replacement.

#### Supported Types

- null and undefined
- boolean
- number, including `NaN` and `Infinity`, which is converted to zero
- string
- RegExp instances
- Date instances (serialized with a call to the Date constructor)

Other value types are serialized with `JSON.stringify`, so functions are not supported. Object and arrays are not tested, please don't use these. 

 
_Example:_

```js
//#set $_STRING = 'foo' + 'bar'
//#set $_NUMBER = 5 - 2
//#set $_DATE   = new Date()
var result = fn($_STRING, $_NUMBER, $_DATE)
```
Output:
```js
var result = fn("foobar", 3, (new Date(1444929015601)))
```

Note the date: Its value is set at the time jspreproc generates the output, and preserved through a call to the `Date` constructor with a fixed value, as there is no standard way to serialize an date object.


### header1

Text to insert before the _top level_ file.  

type   | default | example
-------|---------|----------
string | (none)  | `{headers: '// @module foo\n'}`

As seen in the example, you can use special JavaScript characters like `\n`, `\t`, and so, but can specify an end of line with a caret, too (e.g. `"// @module foo^"`). In the output, any EOLs and carets are replaced with a end of line (as configured by the `--eol-type` option). Use two carets to output a literal one.  

The output of the header is generated with the same engine as the defined symbols but, unlike string values for `#set`, don't enclose the header value in quotes, jspreproc stores this as string.


### headers

Text to insert before each _included_ file.

type   | default | example
-------|---------|---------
string | `\n// __FILE\n\n` | `{headers: "/* __FILE */^"}`

The behavior of this option/property is the same of `--header1`


### indent

type          | default | example
--------------|---------|---------
string/number | (none)    | `{indent: "2s"}` or `{indent: 2}` 

Indentation to add before each line on the included files.  
For a string, the format matches the regex `/$\d+\s*[ts]/`, one or more digits followed by one `t` means tabs and `s` spaces, default for both number or strings is spaces.  
Each level adds indentation.


### eolType

Performs end of line (EOL) normalization.

type | default | example
-----|---------|---------
string: unix, win, mac | `"unix"` | `{eolType: 'win'}`

Converts all EOLs to Unix, Windows, or Mac style. Must Windows editors has no problems handling the default unix `\n` terminator.

**Note:** This normalization is required and can not be disabled.


### emptyLines

Specifies how many consecutive empty lines are preserved in the output.

type | default | example
-----|---------|---------
number | 1 | `{emptyLines: 0}`

There's no range check for this property, only type coercion to integer. Value `0` removes and `-1` preserves all the blank lines, except lines from _conditional comments_, that are removed always.


### comments

Treatment of the comments, both single and multiline.

type | default | example
-----|---------|---------
string: all, none, filter | `"filter"` | `{comments: 'none'}`

Accepted options are `"all"` for keep all comments, `"none"` for remove all, and `"filter"` for apply the filters defined by the `filter` property. Again, _conditional comments_ are always removed.

**Tip:** There's no "single" or "multi" option for remove one of both comment types, but you can do that by creating a _custom filter_. See the [`customFilter`](#customfilter) property for an example. 


### filter

Keep comments matching the specified filter.

type | default | example
-----|---------|---------
string/array: filter name or "all" | `"license"` | `{filter: ['jsdoc','eslint']}`

`filter` can specify one filter, or a list of filters separated with commas as in the example (no spaces please) or an array of strings with multiple filter names.

Predefined jspreproc filters and their regexes are:

- `license` : `/@license\b/`  
   The default and non-removable filter keeps comments with the word "@license" inside.
- [`titles`][titles] : `/^\/(\/\s*|\*[*\s]*)#{1,}/`  
   For markdown titles ala [docco](jashkenas.github.io/docco/ "DOCCO page"), e.g. `// ##`, but multiline too.
- [`jsdoc`][jsdoc]   : `/^\/\*\*[^@]*@[A-Za-z]/`
- [`jslint`][jslint] : `/^\/[*\/](?:jslint|global|property)\b/`
- [`jshint`][jshint] : `/^\/[*\/]\s*(?:jshint|globals|exported)\s/`
- [`eslint`][eslint] : `/^\/[*\/]\s*(?:eslint(?:\s|-[ed])|global\s)/`
- [`jscs`][jscs]     : `/^\/[*\/]\s*jscs:[ed]/`
- [`istanbul`][istanbul] : `/^\/[*\/]\s*istanbul\s/`
- `all` : Enables _**all**_ the filters and yes, this is the easy and slower way.

[titles]: http://daringfireball.net/projects/markdown/ "John Gruber Markdown"
[jsdoc]:  http://usejsdoc.org/ "@use JSDoc"
[jslint]: http://www.jslint.com/ "Douglas Crockford JSLint"
[jshint]: http://jshint.com/ "JSHint site"
[eslint]: http://eslint.org/ "The pluggable linting utility for JavaScript and JSX"
[jscs]:   http://jscs.info/ "JSCS - JavaScript Code Style"
[istanbul]: https://gotwarlost.github.io/istanbul/ "a JS code coverage tool written in JS"


### customFilter

Creates a custom comments filter. 

type | default | example
-----|---------|---------
string/regex<br>(opt array) | -- | `{customFilter: ['@ mod', /^.\*#/]}`

With this option, you instruct to jspreproc for create a regex as a custom filter to apply with `regex.test()` on comments, i.e. the regex must returns `true` to keep the comment.

Custom filters are anonymous and always enabled; you don't need use the `filter` property to enable these.

Common case for custom filters is preserve few comments with special text, but you can use this feature for preserving comments by type as in this examples:

- `"^//"` or `/^\/\//` : preserves single-line comments.
- `"^/**"` or `/^\/\*\*/` : keep multiline, comments blocks starting with two (or more) asterisk characters.
- `"\n *[ \t]@module"` or `/\n\ \*[ \t]@module/` : preserves comment blocks with any line starting with " *" followed by one space or tab and the string "module". 


### version

jspreproc version number as a string.
