# Build BigARTM On Windows (Tested April 18, 2026)

This is an updated Windows build recipe for the latest upstream BigARTM development branch.

Official references:

- Windows developer guide: <https://bigartm.readthedocs.io/en/master/devguide/dev_build_windows.html>
- Upstream repository: <https://github.com/bigartm/bigartm>

Notes:

- As of April 18, 2026, GitHub still shows `master` as the development branch for BigARTM.
- The exact head SHA I built locally was `47e37f982de87aa67bfd475ff1f39da696b181b3` from November 10, 2020.
- The official guide is old. The steps below include the extra fixes needed to build with current Windows tooling.
- This guide was verified on Windows x64 with Visual Studio 2022 Build Tools, CMake, Git, PowerShell, and Python 3.13.

## 1. Install prerequisites

Install these first:

- Git for Windows
- CMake, added to `PATH`
- Visual Studio 2022 Build Tools with Desktop C++ tools
- Python 3.x

Use either:

- `Developer PowerShell for VS 2022`, or
- `x64 Native Tools Command Prompt for VS 2022`

The commands below assume PowerShell.

## 2. Clone BigARTM and vcpkg

```powershell
git clone --branch master https://github.com/bigartm/bigartm.git
cd bigartm
git rev-parse HEAD
```

Optional check: if upstream has not moved, `git rev-parse HEAD` should print `47e37f982de87aa67bfd475ff1f39da696b181b3`.

Now clone `vcpkg` inside the repo:

```powershell
git clone https://github.com/microsoft/vcpkg.git vcpkg
cd vcpkg
.\bootstrap-vcpkg.bat -disableMetrics
.\vcpkg.exe install boost-thread boost-program-options boost-date-time boost-filesystem boost-iostreams boost-system boost-chrono boost-timer boost-uuid --triplet x64-windows-static
cd ..
```

## 3. Apply required source fixes

### `CMakeLists.txt`

Inside the `if (MSVC)` block:

- replace

```cmake
add_definitions("/wd4251")
add_definitions("/MP")
add_definitions("/EHsc")
```

- with

```cmake
add_compile_options($<$<COMPILE_LANGUAGE:C,CXX>:/wd4251>)
add_compile_options($<$<COMPILE_LANGUAGE:C,CXX>:/MP>)
add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/EHsc>)

foreach(flag_var
    CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
    CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO
    CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
    CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
  if (DEFINED ${flag_var})
    string(REPLACE "/MDd" "/MTd" ${flag_var} "${${flag_var}}")
    string(REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
  endif ()
endforeach()
```

### `src/artm/core/helpers.cc`

Add this include:

```cpp
#include "boost/random/mersenne_twister.hpp"
```

### `3rdparty/protobuf-3.0.0/src/google/protobuf/stubs/hash.h`

- change

```cpp
#  define GOOGLE_PROTOBUF_HASH_COMPARE std::hash_compare
```

- to

```cpp
#  define GOOGLE_PROTOBUF_MSC_HASH_CXX11
```

- and change

```cpp
#elif defined(_MSC_VER) && !defined(_STLPORT_VERSION)
```

- to

```cpp
#elif defined(_MSC_VER) && !defined(_STLPORT_VERSION) && \
    !defined(GOOGLE_PROTOBUF_MSC_HASH_CXX11)
```

### `3rdparty/protobuf-3.0.0/src/google/protobuf/repeated_field.h`

Add:

```cpp
#include <cstddef>
```

Change:

```cpp
static const size_t kRepHeaderSize;
```

to:

```cpp
static const size_t kRepHeaderSize = offsetof(Rep, elements);
```

Then remove the out-of-class definition:

```cpp
template<typename Element>
const size_t RepeatedField<Element>::kRepHeaderSize =
    reinterpret_cast<size_t>(&reinterpret_cast<Rep*>(16)->elements[0]) - 16;
```

### `3rdparty/protobuf-3.0.0/src/google/protobuf/compiler/java/java_file.cc`

Change:

```cpp
bool operator ()(const FieldDescriptor* f1, const FieldDescriptor* f2) {
```

to:

```cpp
bool operator ()(const FieldDescriptor* f1, const FieldDescriptor* f2) const {
```

### `python/setup.py`

In `install_requires`, add `six` and pin protobuf to a compatible version:

```python
install_requires=[
    'pandas',
    'numpy',
    'packaging',
    'six',
    'tqdm',
    'protobuf>=3.20.3,<4'
],
```

## 4. Build the native binaries

Set Boost paths from the local `vcpkg` install:

```powershell
$env:VCPKG_ROOT = (Resolve-Path .\vcpkg).Path
$env:BOOST_ROOT = "$env:VCPKG_ROOT\installed\x64-windows-static"
$env:BOOST_INCLUDEDIR = "$env:BOOST_ROOT\include"
$env:BOOST_LIBRARYDIR = "$env:BOOST_ROOT\lib"
```

Configure and build:

```powershell
cmake -S . -B build-nmake-v143-static -G "NMake Makefiles" `
  -DCMAKE_BUILD_TYPE=Release `
  -DBUILD_INTERNAL_PYTHON_API=OFF `
  -DBoost_NO_SYSTEM_PATHS=ON `
  -DBOOST_ROOT="$env:BOOST_ROOT" `
  -DBoost_INCLUDE_DIR="$env:BOOST_INCLUDEDIR" `
  -DBoost_LIBRARY_DIR_RELEASE="$env:BOOST_LIBRARYDIR" `
  -DBoost_USE_STATIC_LIBS=ON

cmake --build build-nmake-v143-static
```

Verify the native build:

```powershell
ctest --test-dir build-nmake-v143-static --output-on-failure
.\build-nmake-v143-static\bin\bigartm.exe --help
```

Expected outputs:

- `build-nmake-v143-static\bin\bigartm.exe`
- `build-nmake-v143-static\bin\artm.dll`
- `build-nmake-v143-static\bin\protoc.exe`

## 5. Build and install the Python API

Create a virtual environment:

```powershell
python -m venv .venv-bigartm
.\.venv-bigartm\Scripts\Activate.ps1
python -m pip install --upgrade pip wheel setuptools
python -m pip install pandas numpy packaging six tqdm protobuf==3.20.3
```

Build the wheel with the native DLL embedded:

```powershell
cd python
$env:ARTM_SHARED_LIBRARY = (Resolve-Path ..\build-nmake-v143-static\bin\artm.dll).Path
..\.venv-bigartm\Scripts\python.exe .\setup.py bdist_wheel --protoc_executable="..\build-nmake-v143-static\bin\protoc.exe" -d .\dist
cd ..
```

Install the wheel:

```powershell
$wheel = Get-ChildItem .\python\dist\*.whl | Select-Object -First 1
.\.venv-bigartm\Scripts\python.exe -m pip install --force-reinstall $wheel.FullName
```

If you use a different Python version, the wheel filename will have a different tag. Install whatever file appears in `python\dist`.

Verify the Python API:

```powershell
.\.venv-bigartm\Scripts\python.exe -c "import artm; print(artm.version())"
```

Expected output:

```text
0.10.1
```

## 6. What to remember

- Native build: use `NMake Makefiles`, not the old Visual Studio generator from the original docs.
- Boost: install it through local `vcpkg` with triplet `x64-windows-static`.
- Python API: build the wheel after the native build and use `protobuf==3.20.3` or another `3.20.x` release.
- The Python package name is `bigartm10`, but the import is `import artm`.

## 7. Quick smoke test

After activation of the venv:

```powershell
python -c "import artm; print(artm.version())"
```

If that prints `0.10.1`, the Python API is ready.
