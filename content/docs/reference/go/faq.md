---
title: "Go FAQ"
weight: 620
toc_hide: false
linkTitle: "FAQ"
no_list: "true"
type: docs
description: "This topic has a list of frequently asked questions about implementing protocol buffers in Go, with answer for each."
---
    

## Versions

### What's the difference between `github.com/golang/protobuf` and `google.golang.org/protobuf`? {#modules}

The
[`github.com/golang/protobuf`](https://pkg.go.dev/github.com/golang/protobuf?tab=overview)
module is the original Go protocol buffer API.

The
[`google.golang.org/protobuf`](https://pkg.go.dev/google.golang.org/protobuf?tab=overview)
module is an updated version of this API designed for simplicity, ease of use,
and safety. The flagship features of the updated API are support for reflection
and a separation of the user-facing API from the underlying implementation.

We recommend that you use `google.golang.org/protobuf` in new code.

Version `v1.4.0` and higher of `github.com/golang/protobuf` wrap the new
implementation and permit programs to adopt the new API incrementally. For
example, the well-known types defined in `github.com/golang/protobuf/ptypes` are
simply aliases of those defined in the newer module. Thus,
[`google.golang.org/protobuf/types/known/emptypb`](https://pkg.go.dev/google.golang.org/protobuf/types/known/emptypb)
and
[`github.com/golang/protobuf/ptypes/empty`](https://pkg.go.dev/github.com/golang/protobuf/ptypes/empty)
may be used interchangeably.

### What are `proto1`, `proto2`, and `proto3`? {#proto-versions}

These are revisions of the protocol buffer *language*. It is different from the
Go *implementation* of protobufs.

*   `proto3` is the current version of the language. This is the most commonly
    used version of the language. We encourage new code to use proto3.

*   `proto2` is an older version of the language. Despite being superseded by
    proto3, proto2 is still fully supported.

*   `proto1` is an obsolete version of the language. It was never released as
    open source.

### There are several different `Message` types. Which should I use? {#message-types}

*   [`"google.golang.org/protobuf/proto".Message`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Message)
    is an interface type implemented by all messages generated by the current
    version of the protocol buffer compiler. Functions that operate on arbitrary
    messages, such as
    [`proto.Marshal`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Marshal)
    or
    [`proto.Clone`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Clone),
    accept or return this type.

*   [`"google.golang.org/protobuf/reflect/protoreflect".Message`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#Message)
    is an interface type describing a reflection view of a message.

    Call the `ProtoReflect` method on a `proto.Message` to get a
    `protoreflect.Message`.

*   [`"google.golang.org/protobuf/reflect/protoreflect".ProtoMessage`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#ProtoMessage)
    is an alias of `"google.golang.org/protobuf/proto".Message`. The two types
    are interchangeable.

*   [`"github.com/golang/protobuf/proto".Message`](https://pkg.go.dev/github.com/golang/protobuf/proto?tab=doc#Message)
    is an interface type defined by the legacy Go protocol buffer API. All
    generated message types implement this interface, but the interface does not
    describe the behavior expected from these messages. New code should avoid
    using this type.

## Common problems

### "`go install`": `working directory is not part of a module` {#working-directory}

On Go 1.15 and below, you have set the environment variable `GO111MODULE=on` and
are running the `go install` command outside of a module directory. Set
`GO111MODULE=auto`, or unset the environment variable.

On Go 1.16 and above, `go install` can be invoked outside of a module by
specifying an explicit version: `go install
google.golang.org/protobuf/cmd/protoc-gen-go@latest`

### `constant -1 overflows protoimpl.EnforceVersion` {#enforce-version}

You are using a generated `.pb.go` file which requires a newer version of the
`"google.golang.org/protobuf"` module.

Update to a newer version with:

```shell
go get -u google.golang.org/protobuf/proto
```

### `undefined: "github.com/golang/protobuf/proto".ProtoPackageIsVersion4` {#enforce-version-apiv1}

You are using a generated `.pb.go` file which requires a newer version of the
`"github.com/golang/protobuf"` module.

Update to a newer version with:

```shell
go get -u github.com/golang/protobuf/proto
```

### What is a protocol buffer namespace conflict? {#namespace-conflict}

All protocol buffers declarations linked into a Go binary are inserted into a
global registry.

Every protobuf declaration (for example, enums, enum values, or messages) has an
absolute name, which is the concatenation of the
[package name](/docs/programming-guides/proto#packages) with
the relative name of the declaration in the `.proto` source file (for example,
`my.proto.package.MyMessage.NestedMessage`). The protobuf language assumes that
all declarations are universally unique.

If two protobuf declarations linked into a Go binary have the same name, then
this leads to a namespace conflict, and it is impossible for the registry to
properly resolve that declaration by name. Depending on which version of Go
protobufs is being used, this will either panic at init-time or silently drop
the conflict and lead to a potential bug later during runtime.

### How do I fix a protocol buffer namespace conflict? {#fix-namespace-conflict}

The way to best fix a namespace conflict depends on the reason why a conflict is
occurring.

Common ways that namespace conflicts occur:

*   **Vendored .proto files.** When a single `.proto` file is generated into two
    or more Go packages and linked into the same Go binary, it conflicts on
    every protobuf declaration in the generated Go packages. This typically
    occurs when a `.proto` file is vendored and a Go package is generated from
    it, or the generated Go package itself is vendored. Users should avoid
    vendoring and instead depend on a centralized Go package for that `.proto`
    file.

    *   If a `.proto` file is owned by an external party and is lacking a
        `go_package` option, then you should coordinate with the owner of that
        `.proto` file to specify a centralized Go package that a plurality of
        users can all depend on.

*   **Missing or generic proto package names.** If a `.proto` file does not
    specify a package name or uses an overly generic package name (for example,
    "my_service"), then there is a high probability that declarations within
    that file will conflict with other declarations elsewhere in the universe.
    We recommend that every `.proto` file have a package name that is
    deliberately chosen to be universally unique (for example, prefixed with the
    name of a company).

    *   Warning: Retroactively changing the package name on a `.proto` file can
        potentially cause the use of extension fields or messages stored in
        `google.protobuf.Any` to stop working properly.

Starting with v1.26.0 of the `google.golang.org/protobuf` module, a hard error
will be reported when a Go program starts up that has multiple conflicting
protobuf names linked into it. While it is preferable that the source of the
conflict be fixed, the fatal error can be immediately worked around in one of
two ways:

1.  **At compile time.** The default behavior for handling conflicts can be
    specified at compile time with a linker-initialized variable: `go build
    -ldflags "-X
    google.golang.org/protobuf/reflect/protoregistry.conflictPolicy=warn"`

2.  **At program execution.** The behavior for handling conflicts when executing
    a particular Go binary can be set with an environment variable:
    `GOLANG_PROTOBUF_REGISTRATION_CONFLICT=warn ./main`

### Why does `reflect.DeepEqual` behave unexpectedly with protobuf messages? {#deepequal}

Generated protocol buffer message types include internal state which can vary
even between equivalent messages.

In addition, the `reflect.DeepEqual` function is not aware of the semantics of
protocol buffer messages, and can report differences where none exist. For
example, a map field containing a `nil` map and one containing a zero-length,
non-`nil` map are semantically equivalent, but will be reported as unequal by
`reflect.DeepEqual`.

Use the
[`proto.Equal`](https://pkg.go.dev/google.golang.org/protobuf/proto#Equal)
function to compare message values.

In tests, you can also use the
[`"github.com/google/go-cmp/cmp"`](https://pkg.go.dev/github.com/google/go-cmp/cmp?tab=doc)
package with the
[`protocmp.Transform()`](https://pkg.go.dev/google.golang.org/protobuf/testing/protocmp#Transform)
option. The `cmp` package can compare arbitrary data structures, and
[`cmp.Diff`](https://pkg.go.dev/github.com/google/go-cmp/cmp#Diff) produces
human-readable reports of the differences between values.

```go
if diff := cmp.Diff(a, b, protocmp.Transform()); diff != "" {
  t.Errorf("unexpected difference:\n%v", diff)
}
```

## Hyrum's Law

### What is Hyrum's Law, and why is it in this FAQ? {#hyrums-law}

[Hyrum's Law](https://www.hyrumslaw.com/) states:

> With a sufficient number of users of an API, it does not matter what you
> promise in the contract: all observable behaviors of your system will be
> depended on by somebody.

A design goal of the latest version of the Go protocol buffer API is to avoid,
where possible, providing observable behaviors that we cannot promise to keep
stable in the future. It is our philosophy that deliberate instability in areas
where we make no promises is better than giving the illusion of stability, only
for that to change in the future after a project has potentially been long
depending on that false assumption.

### Why does the text of errors keep changing? {#unstable-errors}

Tests depending on the exact text of errors are brittle and break often when
that text changes. To discourage unsafe use of error text in tests, the text of
errors produced by this module is deliberately unstable.

If you need to identify whether an error is produced by the
[`protobuf`](https://pkg.go.dev/mod/google.golang.org/protobuf) module, we
guarantee that all errors will match
[`proto.Error`](https://pkg.go.dev/google.golang.org/protobuf/proto?tab=doc#Error)
according to [`errors.Is`](https://pkg.go.dev/errors?tab=doc#Is).

### Why does the output of [`protojson`](https://pkg.go.dev/google.golang.org/protobuf/encoding/protojson) keep changing? {#unstable-json}

We make no promises about the long-term stability of Go's implementation of the
[JSON format for protocol buffers](/docs/programming-guides/proto3#json).
The specification only specifies what is valid JSON, but provides no
specification for a *canonical* format for how a marshaler ought to *exactly*
format a given message. To avoid giving the illusion that the output is stable,
we deliberately introduce minor differences so that byte-for-byte comparisons
are likely to fail.

To gain some degree of output stability, we recommend running the output through
a JSON formatter.

### Why does the output of [`prototext`](https://pkg.go.dev/google.golang.org/protobuf/encoding/prototext) keep changing? {#unstable-text}

We make no promises about the long-term stability of Go's implementation of the
text format. There is no canonical specification of the protobuf text format,
and we would like to preserve the ability to make improvements in the
`prototext` package output in the future. Since we don't promise stability in
the package's output, we've deliberately introduced instability to discourage
users from depending on it.

To obtain some degree of stability, we recommend passing the output of
`prototext` through the
[`txtpbfmt`](https://github.com/protocolbuffers/txtpbfmt) program. The formatter
can be directly invoked in Go using
[`parser.Format`](https://pkg.go.dev/github.com/protocolbuffers/txtpbfmt/parser?tab=doc#Format).

## Miscellaneous

### How do I use a protocol buffer message as a hash key? {#hash}

You need canonical serialization, where the marshaled output of a protocol
buffer message is guaranteed to be stable over time. Unfortunately, no
specification for canonical serialization exists at this time. You'll need to
write your own or find a way to avoid needing one.

### Can I add a new feature to the Go protocol buffer implementation? {#new-feature}

Maybe. We always like suggestions, but we're very cautious about adding new
things.

The Go implementation of protocol buffers strives to be consistent with the
other language implementations. As such, we tend to shy away from feature that
are overly specialized to just Go. Go-specific features hinder the goal of
protocol buffers being a language-neutral data interchange format.

Unless your idea is specific to the Go implementation, you should join the
[protobuf discussion group](http://groups.google.com/group/protobuf) and suggest
it there.

If you have an idea for the Go implementation, file an issue on our issue
tracker:
[https://github.com/golang/protobuf/issues](https://github.com/golang/protobuf/issues)

### Can I add an option to `Marshal` or `Unmarshal` to customize it? {#new-marshal-option}

Only if that option exists in other implementations (e.g., C++, Java). The
encoding of protocol buffers (binary, JSON, and text) must be consistent across
implementations, so a program written in one language is able to read messages
written by another one.

We will not add any options to the Go implementation that affect the data output
by `Marshal` functions or read by `Unmarshal` functions unless an equivalent
option exist in at least one other supported implementation.

### Can I customize the code generated by `protoc-gen-go`? {#custom-code}

In general, no. Protocol buffers are intended to be a language-agnostic data
interchange format, and implementation-specific customizations run counter to
that intent.