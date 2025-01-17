* * *

Fork of htmlparser2@4.1.0 for Joplin.

Patch:

```diff
diff --git a/node_modules/htmlparser2/lib/Parser.js b/node_modules/htmlparser2/lib/Parser.js
index 44b4371..bcd7cc2 100644
--- a/node_modules/htmlparser2/lib/Parser.js
+++ b/node_modules/htmlparser2/lib/Parser.js
@@ -212,6 +212,13 @@ var Parser = /** @class */ (function (_super) {
         this._tagname = "";
     };
     Parser.prototype.onclosetag = function (name) {
+        // When this is true, the onclosetag event will always be emitted
+        // for closing tags (eg </div>) even if that tag was not previously
+        // open. This is needed because we reconstruct the HTML based on
+        // fragments that don't necessarily contain the opening tag.
+        // Without this patch, onopentagname would not be emitted, and
+        // so the closing tag would disappear from the output.
+        var alwaysClose = true;
         this._updatePosition(1);
         if (this._lowerCaseTagNames) {
             name = name.toLowerCase();
@@ -236,11 +243,15 @@ var Parser = /** @class */ (function (_super) {
             else if (name === "p" && !this._options.xmlMode) {
                 this.onopentagname(name);
                 this._closeCurrentTag();
+            } else if (!this._stack.length && alwaysClose) {
+                this._cbs.onclosetag(name);
             }
         }
         else if (!this._options.xmlMode && (name === "br" || name === "p")) {
             this.onopentagname(name);
             this._closeCurrentTag();
+        } else if (!this._stack.length && alwaysClose) {
+            this._cbs.onclosetag(name);
         }
     };
     Parser.prototype.onselfclosingtag = function () {
@@ -331,7 +342,11 @@ var Parser = /** @class */ (function (_super) {
     };
     Parser.prototype.onend = function () {
         if (this._cbs.onclosetag) {
-            for (var i = this._stack.length; i > 0; this._cbs.onclosetag(this._stack[--i]))
+            // Prevent the parser from auto-closing tags. Since we deal with fragments that
+            // maybe contain the opening tag but not the closing one, we don't want that
+            // closing tag to be auto-added.
+            //
+            // for (var i = this._stack.length; i > 0; this._cbs.onclosetag(this._stack[--i]))
                 ;
         }
         if (this._cbs.onend)
```

To fix an HTML parsing issue (tags were allowed to start with non-alphanumeric characters), [this upstream commit](https://github.com/fb55/htmlparser2/commit/bc010de9df09f2d730a69734e05e5175ea8bd2d7) has also been applied.

* * *

# htmlparser2

[![NPM version](http://img.shields.io/npm/v/htmlparser2.svg?style=flat)](https://npmjs.org/package/htmlparser2)
[![Downloads](https://img.shields.io/npm/dm/htmlparser2.svg?style=flat)](https://npmjs.org/package/htmlparser2)
[![Build Status](http://img.shields.io/travis/fb55/htmlparser2/master.svg?style=flat)](http://travis-ci.org/fb55/htmlparser2)
[![Coverage](http://img.shields.io/coveralls/fb55/htmlparser2.svg?style=flat)](https://coveralls.io/r/fb55/htmlparser2)

A forgiving HTML/XML/RSS parser.
The parser can handle streams and provides a callback interface.

## Installation

    npm install htmlparser2

A live demo of htmlparser2 is available [here](https://astexplorer.net/#/2AmVrGuGVJ).

## Usage

```javascript
const htmlparser2 = require("htmlparser2");
const parser = new htmlparser2.Parser(
    {
        onopentag(name, attribs) {
            if (name === "script" && attribs.type === "text/javascript") {
                console.log("JS! Hooray!");
            }
        },
        ontext(text) {
            console.log("-->", text);
        },
        onclosetag(tagname) {
            if (tagname === "script") {
                console.log("That's it?!");
            }
        }
    },
    { decodeEntities: true }
);
parser.write(
    "Xyz <script type='text/javascript'>var foo = '<<bar>>';</ script>"
);
parser.end();
```

Output (simplified):

```
--> Xyz
JS! Hooray!
--> var foo = '<<bar>>';
That's it?!
```

## Documentation

Read more about the parser and its options in the [wiki](https://github.com/fb55/htmlparser2/wiki/Parser-options).

## Get a DOM

The `DomHandler` (known as `DefaultHandler` in the original `htmlparser` module) produces a DOM (document object model) that can be manipulated using the [`DomUtils`](https://github.com/fb55/DomUtils) helper.

The `DomHandler`, while still bundled with this module, was moved to its [own module](https://github.com/fb55/domhandler). Have a look at it for further information.

## Parsing RSS/RDF/Atom Feeds

```javascript
const feed = htmlparser2.parseFeed(content, options);
```

Note: While the provided feed handler works for most feeds, you might want to use [danmactough/node-feedparser](https://github.com/danmactough/node-feedparser), which is much better tested and actively maintained.

## Performance

After having some artificial benchmarks for some time, **@AndreasMadsen** published his [`htmlparser-benchmark`](https://github.com/AndreasMadsen/htmlparser-benchmark), which benchmarks HTML parses based on real-world websites.

At the time of writing, the latest versions of all supported parsers show the following performance characteristics on [Travis CI](https://travis-ci.org/AndreasMadsen/htmlparser-benchmark/builds/10805007) (please note that Travis doesn't guarantee equal conditions for all tests):

```
gumbo-parser   : 34.9208 ms/file ± 21.4238
html-parser    : 24.8224 ms/file ± 15.8703
html5          : 419.597 ms/file ± 264.265
htmlparser     : 60.0722 ms/file ± 384.844
htmlparser2-dom: 12.0749 ms/file ± 6.49474
htmlparser2    : 7.49130 ms/file ± 5.74368
hubbub         : 30.4980 ms/file ± 16.4682
libxmljs       : 14.1338 ms/file ± 18.6541
parse5         : 22.0439 ms/file ± 15.3743
sax            : 49.6513 ms/file ± 26.6032
```

## How does this module differ from [node-htmlparser](https://github.com/tautologistics/node-htmlparser)?

This module started as a fork of the `htmlparser` module.
The main difference is that `htmlparser2` is intended to be used only with node (it runs on other platforms using [browserify](https://github.com/substack/node-browserify)).
`htmlparser2` was rewritten multiple times and, while it maintains an API that's compatible with `htmlparser` in most cases, the projects don't share any code anymore.

The parser now provides a callback interface inspired by [sax.js](https://github.com/isaacs/sax-js) (originally targeted at [readabilitySAX](https://github.com/fb55/readabilitysax)).
As a result, old handlers won't work anymore.

The `DefaultHandler` and the `RssHandler` were renamed to clarify their purpose (to `DomHandler` and `FeedHandler`). The old names are still available when requiring `htmlparser2`, your code should work as expected.

## Security contact information

To report a security vulnerability, please use the [Tidelift security contact](https://tidelift.com/security).
Tidelift will coordinate the fix and disclosure.

## `htmlparser2` for enterprise

Available as part of the Tidelift Subscription

The maintainers of `htmlparser2` and thousands of other packages are working with Tidelift to deliver commercial support and maintenance for the open source dependencies you use to build your applications. Save time, reduce risk, and improve code health, while paying the maintainers of the exact dependencies you use. [Learn more.](https://tidelift.com/subscription/pkg/npm-htmlparser2?utm_source=npm-htmlparser2&utm_medium=referral&utm_campaign=enterprise&utm_term=repo)
