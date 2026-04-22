# Build This Patched BigARTM Repository On Windows (Tested April 18, 2026)

This is an updated Windows build recipe for this patched BigARTM repository.

Official references:

- Windows developer guide: <https://bigartm.readthedocs.io/en/master/devguide/dev_build_windows.html>
- Upstream repository: <https://github.com/bigartm/bigartm>

Notes:

- As of April 18, 2026, GitHub still shows `master` as the development branch for BigARTM.
- The exact head SHA I built locally was `47e37f982de87aa67bfd475ff1f39da696b181b3` from November 10, 2020.
- The official guide is old. This repository already contains the extra fixes needed to build with current Windows tooling.
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

## 2. Clone this patched repository and vcpkg

```powershell
git clone https://github.com/NockLaumOK/bigartm-windows-build.git
cd bigartm-windows-build
git rev-parse HEAD
```

Optional check: `git rev-parse HEAD` should print a commit that includes the modern Windows build fixes. The first patched commit was `2a85668efd7a807fdc654a5ae285fb3c6a43f62c`.

Now clone `vcpkg` inside the repo:

```powershell
git clone https://github.com/microsoft/vcpkg.git vcpkg
cd vcpkg
.\bootstrap-vcpkg.bat -disableMetrics
.\vcpkg.exe install boost-thread boost-program-options boost-date-time boost-filesystem boost-iostreams boost-system boost-chrono boost-timer boost-uuid --triplet x64-windows-static
cd ..
```

Do not manually apply source patches when building from this repository. They are already committed.

## 3. Build the native binaries

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

## 4. Build and install the Python API

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

## 5. What to remember

- Native build: use `NMake Makefiles`, not the old Visual Studio generator from the original docs.
- Boost: install it through local `vcpkg` with triplet `x64-windows-static`.
- Python API: build the wheel after the native build and use `protobuf==3.20.3` or another `3.20.x` release.
- The Python package name is `bigartm10`, but the import is `import artm`.

## 6. Quick smoke test

After activation of the venv:

```powershell
python -c "import artm; print(artm.version())"
```

If that prints `0.10.1`, the Python API is ready.

## 7. What this repository already patches

These fixes are already present in this repository:

- `CMakeLists.txt`: modern MSVC compile-option handling and static runtime flag normalization.
- `src/artm/core/helpers.cc`: explicit Boost random include needed by current Boost/MSVC.
- `3rdparty/protobuf-3.0.0/src/google/protobuf/stubs/hash.h`: avoids obsolete `std::hash_compare` on modern MSVC.
- `3rdparty/protobuf-3.0.0/src/google/protobuf/repeated_field.h`: replaces an old offset calculation with `offsetof`.
- `3rdparty/protobuf-3.0.0/src/google/protobuf/compiler/java/java_file.cc`: makes an STL comparator const-correct.
- `python/setup.py`: adds missing `six` dependency and pins protobuf to the compatible `3.20.x` line.
