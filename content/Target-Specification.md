A **target** specification specifies the language in which reactions are written. This is also the language of the program(s) generated by the Lingua Franca compiler. Every Lingua Franca program begins with a statement of this form:

> **target** *name*;

where *name* gives the name of some Lingua Franca target language. The target languages currently supported are:

- [**C**](writing-reactors-in-c): The target language is C, and the runtime framework is provided by the code generator to be linked with the generated code.
- [**Cpp**](Writing-Reactors-in-C＋＋): The target language is C++.
- [**Python**](Writing-Reactors-in-Python): The target language is Python.
- [**TS**](Writing-Reactors-in-TypeScript): The target language is TypeScript, a typed extension of JavaScript.

A target specification may have optional parameters, the names and values of which depend on which specific target you are using.  The syntax for specifying parameters is:

```
target <targetName> {
    parameterName1: <value>,
    parameterName2: <value>,
    ...
    parameterNameN: <value>
};
```
A comma on the last line is optional.

## Target Parameters

Target parameters are:

- [**build**](#build): A command to execute after code generation instead of the default compile command.
- [**compiler**](#compiler):  A string giving the name of the target language compiler to use.
- [**fast**](#fast): A boolean specifying to execute as fast as possible without waiting for physical time to match logical time.
- [**files**](#files): An array of files to be copied to the directory that contains the generated sources.
- [**flags**](#flags): An arrays of strings giving options to be passed to the target compiler.
- [**keepalive**](#keepalive): A boolean indicate whether to keep executing even if the event queue is empty.
- [**logging**](#logging): An indicator of how much information to print when executing the program.
- [**no-compile**](#no-compile): If true, then do not invoke a target language compiler. Just generate code.
- [**protobufs**](#protobufs): An array of .proto files that are to be compiled and included in the generated code.
- [**timeout**](#timeout): A time value (with units) specifying the logical stop time of execution. See [[Termination]].

Not all targets support all target options, but at a minimum, each target is expected to support **build**, **files**, and **timeout**.
Detailed documentation is given below. 
For example:
```
target C {
    compiler: "cc",
    flags: "-O3",
    fast: true,
    logging: log,
    timeout: 10 secs
};
```
This specifies to use compiler `cc` instead of the default `gcc`, to use optimization level 3, to execute as fast as possible, and to exit execution when logical time has advanced to 10 seconds. Note that all events at logical time 10 seconds greater than the starting logical time will be executed.

A target may support overriding the target parameters on the command line when invoking the compiled program. Refer to the appropriate target language documentation.

Array-valued target parameters are given as a comma-separated list enclosed by `[` and `]`. For example,
```
target C {
    files: ["foo.ext", "bar.ext"]
}
```

### build

A command to execute after code generation instead of the default compile command.  This is either a single string or an array of strings. The specified command(s) will be executed an environment that has the following environment variables defined:

* `LF_CURRENT_WORKING_DIRECTORY`: The directory in which the command is invoked.
* `LF_SOURCE_DIRECTORY`: The directory containing the .lf file being compiled.
* `LF_SOURCE_GEN_DIRECTORY`: The directory in which generated files are placed.
* `LF_BIN_DIRECTORY`: The directory into which to put binaries.

The command will be executed in the same directory as the `.lf` file being compiled. For example, if you specify
```
target C {
    build: "./compile.sh Foo"
}
```
then instead of invoking the C compiler after generating code, the code generator will invoke your `compile.sh` script, which could look something like this:
```
#!/bin/bash
CC="gcc"
$CC ${LF_SOURCE_GEN_DIRECTORY}/$1.c \
    ${LF_SOURCE_GEN_DIRECTORY}/core/platform/lf_macos_support.c \
    -o ${LF_BIN_DIRECTORY}/$1 \
    -pthread -DNUMBER_OF_WORKERS=2
```
which will invoke the `gcc` compiler on the generated file `Foo.c`.

### compiler

A string giving the name of the target language compiler to use.

### fast

A boolean which, if true, specifies to execute as fast as possible without waiting for physical time to match logical time.

### files

An array of files to be copied to the directory that contains the generated sources. These files are given relative to the directory containing the `.lf` file that has the **target** directive. If a file begins with a slash `/`, then the path is assumed to be relative to the root directory of the Lingua Franca source tree.  For example, if you wish to use audio on a Mac, you can specify:
```
target C {
    files: ["/lib/C/util/audio_loop_mac.c", "/lib/C/util/audio_loop.h"]
}
```
Your preamble code can then include these files, for example:
```
preamble {= 
    #include "audio_loop_mac.c"
=}
```
Your reactions can then invoke functions defined in that `.c` file.

Sometimes, you will need access to these files from target code in a reaction. For the C target (at least), the generated program will contain a line like this:
```
    #define TARGET_FILES_DIRECTORY "path"
```
where `path` is the full path to the directory containing these files. This can be used in reactions, for example, to read those files.

### flags

An arrays of strings giving options to be passed to the target compiler. If there is just one flag, this can be an ordinary string.

### keepalive

A boolean value true or false to indicate whether to keep executing even if the event queue is empty. It is particularly useful to set this to true when asynchronous events may trigger a physical action. By default, a program will exit once there are no more events to process.

### logging

By default, when executing a generated Lingua Franca program, error messages, warnings, and informational messages are printed to standard out. You can get additional information printed by setting this parameter to `LOG` or `DEBUG`. The latter is more verbose. Some targets also support [tracing](Tracing), which outputs binary traces of an execution rather than human-readable text and is designed to have minimal impact on performance.

### no-compile

If true, then do not invoke a target language compiler. Just generate code.

### protobufs

An array of .proto files that are to be compiled and included in the generated code.

### timeout

A time value (with units) specifying the logical stop time of execution. See [[Termination]].