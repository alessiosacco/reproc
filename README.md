# reproc <!-- omit in toc -->

**Be sure to select the git tag corresponding to the version you're using to get
the correct documentation. (For example:
<https://github.com/DaanDeMeyer/reproc/tree/v1.0.0> to get the v1.0.0
documentation)**

[![Build Status](https://travis-ci.com/DaanDeMeyer/reproc.svg?branch=master)](https://travis-ci.com/DaanDeMeyer/reproc)
[![Build status](https://ci.appveyor.com/api/projects/status/9d79srq8n7ytnrs5?svg=true)](https://ci.appveyor.com/project/DaanDeMeyer/reproc)
[![Join the chat at https://gitter.im/reproc/Lobby](https://badges.gitter.im/reproc/Lobby.svg)](https://gitter.im/reproc/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

- [What is reproc?](#what-is-reproc)
- [Features](#features)
- [Questions](#questions)
- [Retrieving reproc](#retrieving-reproc)
- [Documentation](#documentation)
- [Error handling](#error-handling)
- [Multithreading](#multithreading)
- [Gotcha's](#gotchas)
- [Avoiding resource leaks](#avoiding-resource-leaks)
  - [POSIX](#posix)
  - [Windows](#windows)
- [Contributing](#contributing)

## What is reproc?

reproc (Redirected Process) is a cross-platform library that starts external
processes from within a C or C++ application, reads/writes to their
stdin/stdout/stderr streams and waits for them to exit or forcefully stops them.

The main use case is working with command line tools from within C or C++
applications.

reproc consists out of two libraries: reproc and reproc++. reproc is a C library
that contains all the low code for interacting with child processes. reproc++
depends on reproc and adapts its API to an idiomatic C++ API. It also adds a few
extra's that make working with child processes from C++ easier.

## Features

- Start any program from within C or C++ code.
- Write to its standard input stream and read from its standard output and
  standard error streams.
- Wait for the program to exit or forcefully stop it yourself. When forcefully
  stopping a process you can optionally allow the process to clean up its
  resources or immediately stop it.
- The core library (reproc) is written in C. An optional C++ wrapper library
  (reproc++) with extra features is available for use in C++ applications.
- Zero dependencies.
- Multiple installation methods. Either build reproc as part of your project or
  install it as a shared library.

## Questions

If you have any questions after reading the readme and documentation you can
either make an issue or ask questions directly in the reproc
[gitter](https://gitter.im/reproc/Lobby) channel.

## Retrieving reproc

There are multiple ways to get reproc into your project. One way is to build
reproc as part of your project using CMake. To do this, we first have to get the
reproc source code into the project. This can be done using any of the following
options:

- If you're using CMake 3.11 or later you can use the CMake `FetchContent` API
  to download reproc when running CMake. See
  <https://cliutils.gitlab.io/modern-cmake/chapters/projects/fetch.html> for an
  example.
- Another option is to include reproc's repository as a git submodule.
  <https://cliutils.gitlab.io/modern-cmake/chapters/projects/submodule.html>
  provides more information.
- A very simple solution is to just include reproc's source code in your
  repository. You can download a zip of the source code without the git history
  and add it to your repository in a separate directory (reproc itself uses the
  `external` directory).

After including reproc's source code in your project, it can be built from the
root CMakeLists.txt file as follows:

```cmake
add_subdirectory(<path-to-reproc>)
```

Options can be specified before calling `add_subdirectory`:

```cmake
set(REPROCXX ON CACHE BOOL "" FORCE)
add_subdirectory(external/reproc)
```

You can also depend on a system installed version of reproc. You can either
build and install reproc to your system yourself or install reproc via a package
manager. reproc is available in the following package repositories:

- Arch User Repository (<https://aur.archlinux.org/packages/reproc>)
- Vcpkg (<https://github.com/Microsoft/vcpkg>)

After installing reproc to the system your build system will have to find it.
reproc provides both CMake config files and pkg-config files to simplify finding
a system installed version of reproc using CMake and pkg-config respectively.
Note that reproc and reproc++ are separate libraries and as a result have
separate config files as well. Make sure to search for the one you need.

Find a system installed reproc using CMake:

```cmake
find_package(reproc) # Find reproc.
find_package(reproc++) # Find reproc++.
```

After building reproc as part of your project or finding a system installed
reproc, you can link against it from within your CMakeLists.txt file as follows:

```cmake
target_link_libraries(myapp reproc::reproc) # Link against reproc.
target_link_libraries(myapp reproc::reproc++) # Link against reproc++.
```

## CMake options

reproc's build can be configured using the following CMake options:

- `REPROCXX`: Build reproc++ (default: `OFF`).
- `REPROC_TESTS`: Build tests (default: `OFF`).
- `REPROC_EXAMPLES`: Build examples (default: `OFF`).
- `REPROC_DOCS`: Build docs (default: `OFF`).
- `REPROC_INSTALL`: Generate install target (default: `ON` unless reproc is
  built as a static library using `add_subdirectory`).

## Documentation

API documentation and examples can be found at
<https://daandemeyer.github.io/reproc/>.

The latest documentation can be built by enabling the cmake `REPROC_DOCS`
option. This requires the latest version of
[Doxygen](https://www.stack.nl/~dimitri/doxygen/) to be installed and available
from the PATH. After building the generated documentation can be found in the
`docs/html` directory of the build directory. Open `docs/html/index.html` in
your browser to view the documentation.

## Error handling

Most functions in reproc's API return `REPROC_ERROR`. The `REPROC_ERROR` enum
represents all possible errors that can occur when calling reproc API functions.
Not all errors apply to each function so the documentation of each function
includes a section detailing which errors can occur. One error that can be
returned by each function that returns `REPROC_ERROR` is `REPROC_UNKNOWN_ERROR`.
`REPROC_UNKNOWN_ERROR` is necessary because the documentation of the underlying
system calls reproc uses doesn't always detail what errors occur in which
circumstances (Windows is especially bad here).

To get more information when a reproc API function returns
`REPROC_UNKNOWN_ERROR` reproc provides the `reproc_system_error` function that
returns the actual system error. Use this function to retrieve the actual system
error and file an issue with the system error and the reproc function that
returned it. With this information an extra value can be added to `REPROC_ERROR`
and you'll be able to check against this value instead of having to check
against `REPROC_UNKNOWN_ERROR`.

reproc++'s API integrates with the C++ standard library error codes mechanism
(`std::error_code` and `std::error_condition`). All functions in reproc++'s API
return `std::error_code` values that contain the actual system error that
occurred. This means the `reproc_system_error` function is not necessary in
reproc++ since the returned error codes stores the actual system error instead
of the enum value in `REPROC_ERROR`. You can still test against these error
codes using the `reproc::errc` error condition enum:

```c++
reproc::process;
std::error_code ec = process.start(...);

if (ec == reproc::errc::file_not_found) {
  std::cerr << "Executable not found. Make sure it is available from the PATH.";
  return 1;
}

if (ec) {
  // Will print the actual system error value from errno or GetLastError() if
  // error is a system error.
  std::cerr << ec.value();
  // You can also print a string representation of the error.
  std::cerr << ec.message();
  return 1;
}
```

Because reproc++ integrates with `std::error_code` you can also test against
reproc++ errors using values from the `std::errc` error condition enum:

```c++
reproc::process;
std::error_code ec = process.start(...);
if (ec == std::errc::not_enough_memory) {
  // handle error
}
```

If needed, you can also convert `std::error_code` values to exceptions using
`std::system_error`:

```c++
reproc::process;
std::error_code ec = process.start(...);
if (ec) { throw std::system_error(ec, "Unable to start process"); }
```

## Multithreading

Guidelines for using a reproc child process from multiple threads:

- Don't wait for or stop the same child process from multiple threads at the
  same time.
- Don't read from or write to the same stream of the same child process from
  multiple threads at the same time.

Different threads can read from or write to different streams at the same time.
This is a valid approach when you want to write to stdin and read from stdout in
parallel.

Look at the [forward](examples/forward.cpp) and
[background](examples/background.cpp) examples to see examples of how to work
with reproc from multiple threads.

## Gotcha's

- (POSIX) On POSIX a parent process is required to wait on a child process that
  has exited (using `reproc_wait`) before all resources related to that process
  can be released by the kernel. If the parent doesn't wait on a child process
  after it exits, the child process becomes a
  [zombie process](https://en.wikipedia.org/wiki/Zombie_process).

- While `reproc_terminate` allows the child process to perform cleanup it is up
  to the child process to correctly clean up after itself. reproc only sends a
  termination signal to the child process. The child process itself is
  responsible for cleaning up its own child processes and other resources.

- When using `reproc_kill` the child process does not receive a chance to
  perform cleanup which could result in resources being leaked. Chief among
  these leaks is that the child process will not be able to stop its own child
  processes. Always try to let a child process exit normally by calling
  `reproc_terminate` before calling `reproc_kill`.

- (Windows) `reproc_kill` is not guaranteed to kill a child process immediately
  on Windows. For more information, read the Remarks section in the
  documentation of the Windows `TerminateProcess` function that reproc uses to
  kill child processes on Windows.

- While reproc tries its very best to avoid leaking file descriptors into child
  processes, there are scenario's where it can't guarantee no file descriptors
  will be leaked to child processes. See
  [Avoiding resource leaks](#avoiding-resource-leaks) for more information.

- (POSIX) Writing to a closed stdin pipe of a child process will crash the
  parent process with the `SIGPIPE` signal. To avoid this the `SIGPIPE` signal
  has to be ignored in the parent process. If the `SIGPIPE` signal is ignored
  `reproc_write` will return `REPROC_STREAM_CLOSED` as expected when writing to
  a closed stdin pipe.

- (POSIX) ignoring the `SIGCHLD` signal by setting its disposition to `SIG_IGN`
  changes the behaviour of the `waitpid` system call which will cause
  `reproc_wait` to stop working as expected. Read the Notes section of the
  `waitpid` man page for more information.

## Avoiding resource leaks

### POSIX

On POSIX systems, by default file descriptors are inherited by child processes
when calling `execve`. To prevent unintended leaking of file descriptors to
child processes, POSIX provides a function `fcntl` which can be used to set the
`FD_CLOEXEC` flag on file descriptors which instructs the underlying system to
close them when `execve` (or one of its variants) is called. However, using
`fcntl` introduces a race condition since any process created in another thread
after a file descriptor is created (for example using `pipe`) but before `fcntl`
is called to set `FD_CLOEXEC` on the file descriptor will still inherit that
file descriptor.

To get around this race condition reproc uses the `pipe2` function (when it is
available) which takes the `O_CLOEXEC` flag as an argument. This ensures the
file descriptors of the created pipe are closed when `execve` is called. Similar
system calls that take the `O_CLOEXEC` flag exist for other system calls that
create file descriptors. If `pipe2` is not available (for example on Darwin)
reproc falls back to calling `fcntl` to set `FD_CLOEXEC` immediately after
creating a pipe.

Darwin does not support the `FD_CLOEXEC` flag on any of its system calls but
instead provides an extra flag for the `posix_spawn` API (a wrapper around
`fork/exec`) named
[POSIX_SPAWN_CLOEXEC_DEFAULT](https://www.unix.com/man-page/osx/3/posix_spawnattr_setflags/)
that instructs `posix_spawn` to close all open file descriptors in the child
process created by `posix_spawn`. However, `posix_spawn` doesn't support
changing the working directory of the child process. A solution to get around
this was implemented in reproc but it was deemed too complex and brittle so it
was removed.

While using `pipe2` prevents file descriptors created by reproc from leaking
into other child processes, file descriptors created outside of reproc without
the `FD_CLOEXEC` flag set will still leak into reproc child processes. To mostly
get around this after forking and redirecting the standard streams (stdin,
stdout, stderr) of the child process we close all file descriptors (except the
standard streams) up to `_SC_OPEN_MAX` (obtained with `sysconf`) in the child
process. `_SC_OPEN_MAX` describes the maximum number of files that a process can
have open at any time. As a result, trying to close every file descriptor up to
this number closes all file descriptors of the child process which includes file
descriptors that were leaked into the child process. However, an application can
manually lower the resource limit at any time (for example with
`setrlimit(RLIMIT_NOFILE)`), which can lead to open file descriptors with a
value above the new resource limit if they were created before the resource
limit was lowered. These file descriptors will not be closed in the child
process since only the file descriptors up to the latest resource limit are
closed. Of course, this only happens if the application manually lowers the
resource limit.

### Windows

On Windows the `CreatePipe` function receives a flag as part of its arguments
that specifies if the returned handles can be inherited by child processes or
not. The `CreateProcess` function also takes a flag indicating whether it should
inherit handles or not. Inheritance for endpoints of a single pipe can be
configured after the `CreatePipe` call using the function
`SetHandleInformation`. A race condition occurs after calling `CreatePipe`
(allowing inheritance) but before calling `SetHandleInformation` in one thread
and calling `CreateProcess` (configured to inherit pipes) in another thread. In
this scenario handles are unintentionally leaked into a child process. We try to
mitigate this in two ways:

- We call `SetHandleInformation` after `CreatePipe` for the handles that should
  not be inherited by any process to lower the chance of them accidentally being
  inherited (just like with `fnctl` if `pipe2` is not available). This only
  works for half of the endpoints created (the ones intended to be used by the
  parent process) since the endpoints intended to be used by the child actually
  need to be inherited by their corresponding child process.
- Windows Vista added the `STARTUPINFOEXW` structure in which we can put a list
  of handles that should be inherited. Only these handles are inherited by the
  child process. This again (just like Darwin's `posix_spawn`) only stops
  reproc's processes from inheriting unintended handles. Other code in an
  application that calls `CreateProcess` without passing a `STARTUPINFOEXW`
  struct containing the handles it should inherit can still unintentionally
  inherit handles meant for a reproc child process. reproc uses the
  `STARTUPINFOEXW` struct if it is available.

## Contributing

When making changes:

- Make sure all code in reproc still compiles by enabling all related CMake
  options:

  ```sh
  cmake -DREPROCXX=ON -DREPROC_TESTS=ON -DREPROC_EXAMPLES=ON ..
  ```

- Format your changes with clang-format and run clang-tidy locally since it will
  run in CI as well.

  If the `REPROC_FORMAT` CMake option is enabled, the reproc-format target is
  added that formats all reproc source files with clang-format. reproc-format is
  added to the `ALL` target so it will automatically run while building.

  If the `REPROC_TIDY` CMake option is enabled, CMake will run clang-tidy on all
  reproc source files while building.

  If CMake can't find clang-format or clang-tidy you can tell it where to look:

  `cmake -DCMAKE_PREFIX_PATH=<clang-install-location> ..`

- Make sure all tests still pass. Tests can be run by executing
  `cmake --build build --target reproc-run-tests` in the root directory of the
  reproc repository.

  However, this method does not allow passing arguments to the tests executable.
  If you want more control over which tests are executed you can run the tests
  executable directly (located at `build/tests/tests`). A list of possible
  options can be found
  [here](https://github.com/onqtam/doctest/blob/master/doc/markdown/commandline.md).

  If you don't have access to every platform, make a pull request and CI will
  compile and run the tests on the platforms you don't have access to.

  Tests will be compiled with sanitizers in CI so make sure to not introduce any
  leaks or undefined behaviour. Enable compiling with sanitizers locally as
  follows:

  `cmake -DREPROC_SANITIZERS=ON ..`

- When adding a new feature, make sure to implement it for both POSIX and
  Windows.
- When adding a new feature, add a new test for it or modify an existing one to
  also test the new feature.
- Make sure to update the relevant documentation if needed or write new
  documentation.
- Make sure to follow the .editorconfig rules. Most editors have an editorconfig
  plugin to make this easy.
- reproc doesn't store a .gitignore file in the repository so everyone can add a
  custom .gitignore file locally. Example .gitignore file:

  ```txt
  # Exclude the .gitignore file itself so it isn't pushed to Github.
  .gitignore

  # Exclude the build directory.
  build/
  ```
