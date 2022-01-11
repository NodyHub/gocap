# A take on supply chain security in Go

List your dependencies capabilities and monitor if updates require more 
capabilities.

## The Problem

> Every imported package gives that package's author basically remote code
> execution for your software.

## The Idea

A [video on WASI](https://www.youtube.com/watch?v=fh9WXPu0hw8) by
[linclark](https://twitter.com/linclark) brought me to the idea how cool it would be if we could pass permissions to our
dependencies.

In Go this could look something like this:

```
module github.com/cugu/gocap

go 1.20

require (
	github.com/go-chi/chi       v5.0.7   (network:read)
	github.com/mattn/go-sqlite3 v1.14.10 (file:read, file:write)
	github.com/sirupsen/logrus  v1.8.1   (os:stdout)
	github.com/yuin/goldmark    v1.4.4   ()
)
```

`chi` would be able to receive network requests, `go-sqlite3` would be able to read and write files and `logrus` could
write to stdout. But also all those modules would be limited to those capabilities and, for example, the logging
library `logrus` would not be able to interact with files, the network or execute code.

Changes of dependencies would be much less critical in many cases, as a potential attacker would have only limited
attack surface besides stealing your CPU cycles.

## A simpler but working approach: GoCap

Implementing the approach above would require changes to Go itself. So I came up with another, simpler approach: GoCap.
GoCap can check and validate the source code of dependencies for their capabilities and is ment to be included into the
testing phase of the build process. This way GoCap can to at least pin the capabilities of dependencies.

GoCap provides simple capability checking for Go using a `go.cap` file. The
`go.cap` files lists all package dependencies that require critical permissions like file access, execution rights or
network access. Unlike the idea above GoCap works on packages not modules and capabilities are based on the imported
packages of the standard library.

The `go.cap` file for GoCap itself looks like this:

```
github.com/cugu/gocap (execute, file)

github.com/alecthomas/kong (file, syscall)
github.com/pkg/errors (runtime)
```

### gocap generate

`gocap generate <path>` prints a valid `go.cap` file. It lists all dependency packages that require critical permissions
like file access, execution rights or network access.

*Example*

```shell
gocap generate github.com/cugu/gocap > go.cap
```

*go.cap*

```
github.com/cugu/gocap (execute, file)

github.com/alecthomas/kong (file, syscall)
github.com/pkg/errors (runtime)
``` 

### gocap check

`gocap check <path>` compares a local `go.cap` file with the actual required capabilities by dependency packages. Any
missmatch results in a non-zero exit code, so you can use GoCap check in your CI pipelines.

*Example*

```shell
gocap check .
```

*Output*

```
github.com/alecthomas/kong
	capability 'syscall' not provided by go.cap file, add to go.cap file if you want to grant the capability
github.com/pkg/errors
	unnecessary capability 'network', please remove from go.cap file
``` 