# TinyGo WebAssembly examples

The examples here show two different ways of using WebAssembly with TinyGo;

1. Defining and exporting functions via the `//go:export <name>` directive. See
[the export folder](./export) for an example of this.
1. Defining and executing a `func main()`. This is similar to how the Go
standard library implementation works. See [the main folder](./main) for an
example of this.

## Building

Build using the `tinygo` compiler:

```bash
$ tinygo build -o ./wasm.wasm -target wasm ./main/main.go
```

This creates a `wasm.wasm` file, which we can load in JavaScript and execute in
a browser.

This examples folder contains two examples that can be built using `make`:

```bash
$ make export
```

```bash
$ make main
```

## Running

Start the local webserver:

```bash
$ go run main.go
Serving ./html on http://localhost:8080
```

`fmt.Println` prints to the browser console.

## How it works

Execution of the contents require a few JS helper functions which are called
from WebAssembly. We have defined these in
[wasm_exec.js](../../../targets/wasm_exec.js). It is based on
`$GOROOT/misc/wasm/wasm_exec.js` from the standard library, but is slightly
different. Ensure you are using the same version of `wasm_exec.js` as the
version of `tinygo` you are using to compile.

The general steps required to run the WebAssembly file in the browser includes
loading it into JavaScript with `WebAssembly.instantiateStreaming`, or
`WebAssembly.instantiate` in some browsers:

```js
const go = new Go(); // Defined in wasm_exec.js
const WASM_URL = 'wasm.wasm';

var wasm;

if ('instantiateStreaming' in WebAssembly) {
	WebAssembly.instantiateStreaming(fetch(WASM_URL), go.importObject).then(function (obj) {
		wasm = obj.instance;
		go.run(wasm);
	})
} else {
	fetch(WASM_URL).then(resp =>
		resp.arrayBuffer()
	).then(bytes =>
		WebAssembly.instantiate(bytes, go.importObject).then(function (obj) {
			wasm = obj.instance;
			go.run(wasm);
		})
	)
}
```

If you have used explicit exports, you can call them by invoking them under the
`wasm.exports` namespace. See the [`export`](./export/wasm.js) directory for an
example of this.

In addition to this piece of JavaScript, it is important that the file is served
with the correct `Content-Type` header set.

```go
package main

import (
	"log"
	"net/http"
	"strings"
)

const dir = "./html"

func main() {
	fs := http.FileServer(http.Dir(dir))
	log.Print("Serving " + dir + " on http://localhost:8080")
	http.ListenAndServe(":8080", http.HandlerFunc(func(resp http.ResponseWriter, req *http.Request) {
		if strings.HasSuffix(req.URL.Path, ".wasm") {
			resp.Header().Set("content-type", "application/wasm")
		}

		fs.ServeHTTP(resp, req)
	}))
}
```

This simple server serves anything inside the `./html` directory on port `8080`,
setting any `*.wasm` files `Content-Type` header appropriately.
