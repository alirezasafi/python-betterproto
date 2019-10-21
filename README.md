# Better Protobuf / gRPC Support for Python

This project aims to provide an improved experience when using Protobuf / gRPC in a modern Python environment by making use of modern language features and generating readable, understandable, idiomatic Python code. It will not support legacy features or environments. The following are supported:

- Protobuf 3 & gRPC code generation
  - Both binary & JSON serialization is built-in
- Python 3.7+ making use of:
  - Enums
  - Dataclasses
  - `async`/`await`
  - Relative imports
  - Mypy type checking

This project is heavily inspired by, and borrows functionality from:

- https://github.com/protocolbuffers/protobuf/tree/master/python
- https://github.com/eigenein/protobuf/
- https://github.com/vmagamedov/grpclib

## Motivation

This project exists because I am unhappy with the state of the official Google protoc plugin for Python.

- No `async` support (requires additional `grpclib` plugin)
- No typing support or code completion/intelligence (requires additional `mypy` plugin)
- No `__init__.py` module files get generated
- Output is not importable
  - Import paths break in Python 3 unless you mess with `sys.path`
- Bugs when names clash (e.g. `codecs` package)
- Generated code is not idiomatic
  - Completely unreadable runtime code-generation
  - Much code looks like C++ or Java ported 1:1 to Python
  - Capitalized function names like `HasField()` and `SerializeToString()`
  - Uses `SerializeToString()` rather than the built-in `__bytes__()`

This project is a reimplementation from the ground up focused on idiomatic modern Python to help fix some of the above. While it may not be a 1:1 drop-in replacement due to changed method names and call patterns, the wire format is identical.

## Installation & Getting Started

First, install the package:

```sh
$ pip install betterproto
```

Now, given a proto file, e.g `example.proto`:

```protobuf
syntax = "proto3";

package hello;

// Greeting represents a message you can tell a user.
message Greeting {
  string message = 1;
}
```

You can run the following:

```sh
$ protoc -I . --python_betterproto_out=. example.proto
```

This will generate `hello.py` which looks like:

```py
# Generated by the protocol buffer compiler.  DO NOT EDIT!
# sources: hello.proto
# plugin: python-betterproto
from dataclasses import dataclass

import betterproto


@dataclass
class Hello(betterproto.Message):
    """Greeting represents a message you can tell a user."""

    message: str = betterproto.string_field(1)
```

Now you can use it!

```py
>>> from hello import Hello
>>> test = Hello()
>>> test
Hello(message='')

>>> test.message = "Hey!"
>>> test
Hello(message="Hey!")

>>> serialized = bytes(test)
>>> serialized
b'\n\x04Hey!'

>>> another = Hello().parse(serialized)
>>> another
Hello(message="Hey!")

>>> another.to_dict()
{"message": "Hey!"}
>>> another.to_json(indent=2)
'{\n  "message": "Hey!"\n}'
```

### Async gRPC Support

The generated Protobuf `Message` classes are compatible with [grpclib](https://github.com/vmagamedov/grpclib) so you are free to use it if you like. That said, this project also includes support for async gRPC stub generation with better static type checking and code completion support. It is enabled by default.

Given an example like:

```protobuf
syntax = "proto3";

package echo;

message EchoRequest {
  string value = 1;
  // Number of extra times to echo
  uint32 extra_times = 2;
}

message EchoResponse {
  repeated string values = 1;
}

message EchoStreamResponse  {
  string value = 1;
}

service Echo {
  rpc Echo(EchoRequest) returns (EchoResponse);
  rpc EchoStream(EchoRequest) returns (stream EchoStreamResponse);
}
```

You can use it like so (enable async in the interactive shell first):

```py
>>> import echo
>>> from grpclib.client import Channel

>>> channel = Channel(host="127.0.0.1", port=1234)
>>> service = echo.EchoStub(channel)
>>> await service.echo(value="hello", extra_times=1)
EchoResponse(values=["hello", "hello"])

>>> async for response in service.echo_stream(value="hello", extra_times=1)
    print(response)

EchoStreamResponse(value="hello")
EchoStreamResponse(value="hello")
```

### JSON

Both serializing and parsing are supported to/from JSON and Python dictionaries using the following methods:

- Dicts: `Message().to_dict()`, `Message().from_dict(...)`
- JSON: `Message().to_json()`, `Message().from_json(...)`

### Determining if a message was sent

Sometimes it is useful to be able to determine whether a message has been sent on the wire. This is how the Google wrapper types work to let you know whether a value is unset, set as the default (zero value), or set as something else, for example.

Use `Message().serialized_on_wire` to determine if it was sent. This is a little bit different from the official Google generated Python code:

```py
# Old way (official Google Protobuf package)
>>> mymessage.HasField('myfield')

# New way (this project)
>>> mymessage.myfield.serialized_on_wire
```

## Development

First, make sure you have Python 3.7+ and `pipenv` installed:

```sh
# Get set up with the virtual env & dependencies
$ pipenv install --dev

# Link the local package
$ pipenv shell
$ pip install -e .
```

### Tests

There are two types of tests:

1. Manually-written tests for some behavior of the library
2. Proto files and JSON inputs for automated tests

For #2, you can add a new `*.proto` file into the `betterproto/tests` directory along with a sample `*.json` input and it will get automatically picked up.

Here's how to run the tests.

```sh
# Generate assets from sample .proto files
$ pipenv run generate

# Run the tests
$ pipenv run tests
```

### TODO

- [x] Fixed length fields
  - [x] Packed fixed-length
- [x] Zig-zag signed fields (sint32, sint64)
- [x] Don't encode zero values for nested types
- [x] Enums
- [x] Repeated message fields
- [x] Maps
  - [x] Maps of message fields
- [x] Support passthrough of unknown fields
- [x] Refs to nested types
- [x] Imports in proto files
- [x] Well-known Google types
- [ ] OneOf support
  - [x] Basic support on the wire
  - [ ] Check which was set from the group
  - [ ] Setting one unsets the others
- [ ] JSON that isn't completely naive.
  - [x] 64-bit ints as strings
  - [x] Maps
  - [x] Lists
  - [x] Bytes as base64
  - [ ] Any support
  - [x] Enum strings
  - [ ] Well known types support (timestamp, duration, wrappers)
- [ ] Async service stubs
  - [x] Unary-unary
  - [x] Server streaming response
  - [ ] Client streaming request
- [ ] Renaming messages and fields to conform to Python name standards
- [ ] Renaming clashes with language keywords and standard library top-level packages
- [x] Python package
- [ ] Automate running tests
- [ ] Cleanup!

## License

Copyright © 2019 Daniel G. Taylor

http://dgt.mit-license.org/