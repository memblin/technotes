# Python Debug Builds

It is sometimes necessary to build a pydebug enabled Python interpreter for debugging purposes.

The documentation even recommends always developing under a pydebug build unless you're taking performance measurements.

> CPython provides several compilation flags which help with debugging various things. While all of the known flags can be found in the [Misc/SpecialBuilds.txt](https://github.com/python/cpython/blob/main/Misc/SpecialBuilds.txt) file, the most critical one is the `Py_DEBUG` flag which creates what is known as a “pydebug” build. This flag turns on various extra sanity checks which help catch common issues. The use of the flag is so common that turning on the flag is a basic compile option.
>
> You should always develop under a pydebug build of CPython (the only instance of when you shouldn’t is if you are taking performance measurements). Even when working only on pure Python code the pydebug build provides several useful checks that one should not skip.

ref: [Python Developers Guide: Compile and build](https://devguide.python.org/getting-started/setup-building/#compile-and-build)

> [!IMPORTANT]
> IMPORTANT: if you want to build a debug-enabled Python, it is recommended that
you use ``./configure --with-pydebug``, rather than the options listed in `Misc/SpecialBuilds.txt`.

Once we have our pydebug build we can create purpose dedicated venvs for development and testing needs.

## Requirements

System preperation for building python with debug enabled.

- [Debian 11](python-build-reqs-debian11.md)

## Configure and Build

Once the system has the requirements installed we can configure and build.

The various configure and build flags are documented in

- https://docs.python.org/dev/using/configure.html

### Interesting options

[--enable-pystats](https://docs.python.org/dev/using/configure.html#cmdoption-enable-pystats): Turn on internal Python performance statistics gathering.

[--prefix=PREFIX](): Install architecture-independent files in PREFIX. On Unix, it defaults to /usr/local.

[--without-pymalloc](https://docs.python.org/dev/using/configure.html#cmdoption-without-pymalloc): Disable the specialized Python memory allocator pymalloc (enabled by default).

[--enable-profiling](https://docs.python.org/dev/using/configure.html#cmdoption-enable-profiling): Enable C-level code profiling with gprof (disabled by default).

[--with-pydebug](https://docs.python.org/dev/using/configure.html#cmdoption-with-pydebug): [Build Python in debug mode](https://docs.python.org/dev/using/configure.html#debug-build): define the Py_DEBUG macro (disabled by default).

[--with-trace-refs](https://docs.python.org/dev/using/configure.html#cmdoption-with-trace-refs): Enable tracing references for debugging purpose (disabled by default).

[--with-valgrind](https://docs.python.org/dev/using/configure.html#cmdoption-with-valgrind): Enable Valgrind support (default is no). (See: [README.valgrind](https://github.com/python/cpython/blob/main/Misc/README.valgrind))

[--with-address-sanitizer](https://docs.python.org/dev/using/configure.html#cmdoption-with-address-sanitizer): Enable AddressSanitizer memory error detector, `asan` (default is no). [3.6]

[--with-memory-sanitizer](https://docs.python.org/dev/using/configure.html#cmdoption-with-memory-sanitizer): Enable MemorySanitizer allocation error detector, `msan` (default is no). [3.6]

[--with-thread-sanitizer](https://docs.python.org/dev/using/configure.html#cmdoption-with-thread-sanitizer): Enable ThreadSanitizer data race detector, `tsan` (default is no). [3.13]

### Typical Configuration

#### Checkout the version tag you need

```bash
# Change directory to the repo
cd ~/repos/github/python/cpython

# Checkout the version you need to build
git checkout -b v3.9.2 tags/v3.9.2

# Check in the git log that you got what you wanted
# Expect last commit to show the version like Python 3.9.2 in this case
git log
```

#### Configure the code

##### gcc

```bash
# Create a build subdir and change directory into it
mkdir build-v3.9.2-gcc-debug && cd $_

# Configure the build
../configure --with-pydebug --prefix=/opt/python/v3.9.2-gcc-debug
```

##### clang

```bash
# Create a build subdir and change directory into it
mkdir build-v3.9.2-clang-debug && cd $_

# Configure the build
export CC=/usr/bin/clang
export CFLAGS='-Wno-unused-value -Wno-empty-body -Qunused-arguments'
../configure --with-pydebug --prefix=/opt/python/v3.9.2-clang-debug
```

#### Build & Test

```bash
# Build
make -s -j2

# Test
make test
```

#### Using the built Python interpreter

We can use the python interpreter directly or by building venvs.

```bash
# The built python interpreter is in the build dir
./python --version

# Create a venv that uses the built interpreter in the build dir
./python -m venv venv

# Activate it
source venv/bin/activate
```

We can also install the built interpreter and then create venvs from the globally available customized interpreter for testing needs.

```bash
# Package can be installed if desired, it will install into the --prefix location
# sudo may be required depending on install location and user perms
sudo make install

# Create venvs using the installed v3.9.2-clang-debug python interpreter
#
# Global shared venv
sudo /opt/python/v3.9.2-gcc-debug/bin/python3 -m venv /opt/venvs/salt-v3.9.2-gcc-debug
# or
sudo /opt/python/v3.9.2-clang-debug/bin/python3 -m venv /opt/venvs/salt-v3.9.2-clang-debug

# User venv
/opt/python/v3.9.2-gcc-debug/bin/python3 -m venv ~/.venvs/salt-v3.9.2-gcc-debug
# or
/opt/python/v3.9.2-clang-debug/bin/python3 -m venv ~/.venvs/salt-v3.9.2-clang-debug
```

#### Using Python venvs

```bash
# Activate the venv
# Use the path to the venv you want to source the appropriate activate script
source venv/bin/activate

# Upgrade pip in the venv unless you need a specific version of pip for your testing
pip install --upgrade pip
```

This example uses a global venv to install salt which needs to run as root for our current debugging purposes.

```bash
# su to root
sudo su -

# As root, activate the target venv
source /opt/venvs/salt-v3.9.2-gcc-debug/bin/activate

# Upgrade pip
pip install --upgrade pip

# Install salt into the venv from the v3006.6-adelaide tag in the repos
# The example is for a local clone of a remote to save hitting github repeatedly. One
# can also use git+https:// and git+ssh:// URI designators if desired.
pip install git+file:///home/tkcadmin/repos/github/knotsalt/salt@v3006.6-adelaide

# Run salt-master in foreground with debug logging using salt-v3.9.2-gcc-debug
# assuming the venv is active
salt-master -l debug
#
# If the venv is not active you can run things from the venv directly.
# This can be helpful for scripts or service unit files and similar use-cases.
/opt/venvs/salt-v3.9.2-gcc-debug/bin/salt-master -l debug
#
# Salt configs would need to be added to /etc/salt to test specific use-case issues
```

## Python Development Mode

The Python Development Mode introduces additional runtime checks that are too expensive to be enabled by default. It should not be more verbose than the default if the code is correct; new warnings are only emitted when an issue is detected.

ref: [Python Development Mode](https://docs.python.org/dev/library/devmode.html#devmode)

## References

- https://devguide.python.org/getting-started/setup-building/
