# Building OpenMoonRay on Ubuntu 22.04

> **Unofficial community build guide**
>
> This document records the process I used to successfully build
> OpenMoonRay from source on Ubuntu 22.04.5 LTS.
>
> OpenMoonRay's tested Linux environment is Rocky Linux 9. Ubuntu
> required several dependency, ABI, and linker adjustments. This is not
> an official OpenMoonRay-supported Ubuntu configuration.

## Final Result

At the end of this process I was able to:

-   Configure OpenMoonRay on Ubuntu 22.04.
-   Build the complete source tree to 100%.
-   Build `moonray_rendering_pbr_tests`.
-   Launch the `moonray` executable and run `--help`.
-   Keep the upstream MoonRay source tree completely clean.

Final Git status:

``` text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

This means the compatibility work described here was performed through
dependencies and build configuration rather than by modifying tracked
OpenMoonRay source files.

------------------------------------------------------------------------

## 1. System Used

``` text
OS:              Ubuntu 22.04.5 LTS
Architecture:    x86_64
Compiler:        GCC/G++ 11.4.0
System Python:   Python 3.10.12
GPU:             NVIDIA GeForce GTX 1070 Mobile
NVIDIA Driver:   580.126.09
CPU:             Intel Core i7-6700HQ
Logical CPUs:    8
```

The CPU supports AVX, AVX2, FMA, BMI1, and BMI2, but not AVX-512. I
therefore configured Embree around AVX2 rather than AVX-512.

This was initially a CPU-oriented, non-GUI build:

``` text
-DMOONRAY_USE_OPTIX=NO
-DBUILD_QT_APPS=NO
```

------------------------------------------------------------------------

## 2. Avoiding the Conda Toolchain

My machine also had Anaconda installed. Inside the Conda environment I
had Python 3.12.4 and CMake 3.29.4.

This caused problems because CMake could discover libraries from the
Anaconda environment rather than the Ubuntu/system or custom dependency
installations.

I therefore left Conda:

``` bash
conda deactivate
```

For this build I explicitly used Ubuntu's system Python:

``` text
/usr/bin/python3
```

which was Python 3.10.12.

I also avoided the CMake executable installed through Anaconda.

------------------------------------------------------------------------

## 3. CMake

Ubuntu 22.04 provided CMake 3.22.1, which was too old for the
configuration I needed.

Instead of replacing Ubuntu's system CMake, I installed a standalone
Kitware CMake. The successful version was:

``` text
CMake 3.31.12
```

Its executable was:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake
```

This avoided modifying the system CMake installation.

------------------------------------------------------------------------

## 4. General Ubuntu Development Dependencies

During the process I installed Ubuntu development packages needed by
MoonRay and its dependencies, including tools/libraries for:

-   Bison and Flex
-   wget and patch
-   Python development headers and pybind11
-   Boost
-   Lua 5.4
-   Blosc and zlib
-   log4cplus
-   CppUnit
-   libmicrohttpd
-   JsonCpp
-   Wayland and XKB
-   X11
-   OpenGL and image libraries
-   Git LFS

One important package discovered later was `lua5.4`, because the first
MoonRay build failed when the Lua executable/compiler was unavailable.

Package names can vary between distributions and Ubuntu releases, so
this guide documents the environment I used rather than claiming a
universal package list.

------------------------------------------------------------------------

## 5. Repository Layout

I kept MoonRay itself separate from all manually built dependencies.

MoonRay repository:

``` text
~/Documents/OpenSource/openmoonray
```

Dependency workspace:

``` text
~/Documents/OpenSource/moonray-deps
```

The layout eventually looked approximately like:

``` text
Documents/
└── OpenSource/
    ├── openmoonray/
    │   ├── moonray/
    │   └── build/
    │
    └── moonray-deps/
        ├── TBB/
        ├── tbb-install/
        ├── OpenSubdiv/
        ├── opensubdiv-install/
        ├── USD/
        ├── usd-install/
        ├── OpenEXR/
        ├── openexr-install/
        ├── OpenColorIO/
        ├── openocio-install/
        ├── OpenImageIO/
        ├── openimageio-build/
        ├── openimageio-install/
        ├── OpenVDB/
        ├── openvdb-install/
        ├── Embree/
        ├── embree-install/
        ├── OpenImageDenoise/
        ├── openimagedenoise-install/
        ├── Random123/
        └── ispc-v1.21.0-linux/
```

Keeping dependency source, build, and install trees separate was
extremely helpful.

------------------------------------------------------------------------

## 6. Clone OpenMoonRay

I cloned OpenMoonRay recursively so its Git submodules were available:

``` bash
git clone --recurse-submodules https://github.com/OpenMoonRay/openmoonray.git
```

The repository remained on `main` during the dependency and build-system
work. I deliberately did not create my feature branch yet because I
wanted a working baseline before modifying MoonRay.

------------------------------------------------------------------------

## 7. Why I Did Not Use the Rocky Linux Preset Directly

OpenMoonRay contains presets oriented toward Rocky Linux, including:

``` text
rocky9-release
rocky9-houdini-release
```

Because this machine runs Ubuntu 22.04, I created an explicit CMake
configuration instead of assuming the Ubuntu environment was identical
to Rocky Linux.

The dependency versions were kept close to those used by the project's
Rocky Linux build environment.

------------------------------------------------------------------------

## 8. Dependency Versions

  Dependency         Version
  ------------------ ----------
  TBB                2020.3.3
  OpenSubdiv         3.5.0
  USD                23.08
  OpenEXR            3.1.8
  Imath              3.1.x
  OpenColorIO        2.2.1
  OpenImageIO        2.4.8.0
  ISPC               1.21.0
  OpenImageDenoise   2.3.3
  Embree             4.2.0
  Random123          1.14.0
  OpenVDB            9.1.0

The particularly important compatibility decision was to use **TBB
2020.3 consistently** rather than mixing it with Ubuntu's newer oneTBB.

------------------------------------------------------------------------

## 9. TBB 2020.3.3

I built an isolated copy of TBB 2020.3.3 and installed it under:

``` text
~/Documents/OpenSource/moonray-deps/tbb-install
```

This was necessary because USD 23.08 and other dependencies in the
MoonRay stack expect the older TBB API/ABI.

Ubuntu 22.04 also provides newer oneTBB packages. Allowing some
dependencies to use those while others used TBB 2020 eventually caused
linker problems.

------------------------------------------------------------------------

## 10. OpenSubdiv 3.5.0

I built OpenSubdiv 3.5.0 from tag:

``` text
v3_5_0
```

and installed it to:

``` text
~/Documents/OpenSource/moonray-deps/opensubdiv-install
```

It was configured against the isolated TBB installation rather than
Ubuntu's oneTBB.

------------------------------------------------------------------------

## 11. USD 23.08

I built USD 23.08 from tag:

``` text
v23.08
```

USD was built using Python 3.10, Boost 1.74, TBB 2020.3, and OpenSubdiv
3.5.0, and installed to:

``` text
~/Documents/OpenSource/moonray-deps/usd-install
```

This supplied MoonRay's required `pxrConfig.cmake`.

One of the early MoonRay configuration blockers was that CMake could not
locate `pxrConfig.cmake`. Building USD separately and pointing MoonRay
at its installation resolved this.

The important MoonRay option eventually became:

``` bash
-Dpxr_DIR=$HOME/Documents/OpenSource/moonray-deps/usd-install
```

with the USD install also present in `CMAKE_PREFIX_PATH`.

------------------------------------------------------------------------

## 12. OpenEXR 3.1.8

I built OpenEXR 3.1.8 from tag:

``` text
v3.1.8
```

and installed it to:

``` text
~/Documents/OpenSource/moonray-deps/openexr-install
```

This build also provided a compatible Imath 3.1 installation.

For this setup, OpenEXR was built as a custom dependency rather than
relying solely on Ubuntu's OpenEXR packages.

------------------------------------------------------------------------

## 13. OpenColorIO 2.2.1

I built OpenColorIO 2.2.1 and installed it to:

``` text
~/Documents/OpenSource/moonray-deps/openocio-install
```

It was built as shared libraries.

------------------------------------------------------------------------

## 14. OpenImageIO 2.4.8.0

I built OpenImageIO 2.4.8.0 from tag:

``` text
v2.4.8.0
```

and installed it to:

``` text
~/Documents/OpenSource/moonray-deps/openimageio-install
```

Initially OIIO detected Ubuntu's newer TBB 2021.5. This appeared to work
during the OIIO build itself, but later caused trouble when MoonRay
linked libraries using different TBB ABIs.

OIIO was eventually rebuilt against TBB 2020.3, as documented below.

------------------------------------------------------------------------

## 15. ISPC 1.21.0

I used the prebuilt Linux distribution of ISPC 1.21.0.

The compiler executable was:

``` text
~/Documents/OpenSource/moonray-deps/ispc-v1.21.0-linux/bin/ispc
```

MoonRay was configured explicitly with:

``` bash
-DCMAKE_ISPC_COMPILER=$HOME/Documents/OpenSource/moonray-deps/ispc-v1.21.0-linux/bin/ispc
```

This is important because MoonRay contains both C++ and ISPC rendering
code.

------------------------------------------------------------------------

## 16. OpenImageDenoise 2.3.3

I built OpenImageDenoise 2.3.3 from tag:

``` text
v2.3.3
```

and installed it to:

``` text
~/Documents/OpenSource/moonray-deps/openimagedenoise-install
```

It was configured using the isolated TBB and ISPC environment.

Its weights submodule was required:

``` bash
git submodule update --init --recursive
```

------------------------------------------------------------------------

## 17. Embree 4.2.0

I built Embree 4.2.0 from tag:

``` text
v4.2.0
```

and installed it to:

``` text
~/Documents/OpenSource/moonray-deps/embree-install
```

The Rocky Linux recipe allowed AVX-512, but the i7-6700HQ used for this
Ubuntu build does not support AVX-512. Therefore this build was
configured around AVX2.

Embree was also tied to the isolated TBB environment.

------------------------------------------------------------------------

## 18. Random123 1.14.0

I checked out Random123 1.14.0 from tag:

``` text
v1.14.0
```

Random123 is header-only.

Its source directory was:

``` text
~/Documents/OpenSource/moonray-deps/Random123
```

Before configuring MoonRay in a fresh shell I exported:

``` bash
export RANDOM123_ROOT=$HOME/Documents/OpenSource/moonray-deps/Random123
```

------------------------------------------------------------------------

## 19. OpenVDB: Initial System Version Problem

Ubuntu provided OpenVDB 8.1, and initially the build used that system
OpenVDB.

At approximately 76% of the MoonRay build, compilation reached
`PathIntegrator.cc` and failed because the system OpenVDB stack brought
in an incompatible version of OpenEXR/Imath.

The build was effectively mixing custom OpenEXR/Imath 3.1 with Ubuntu
OpenVDB 8.1 and its system OpenEXR/Imath dependencies.

The resulting `half`/Imath incompatibility prevented compilation.

------------------------------------------------------------------------

## 20. Build OpenVDB 9.1.0

Instead of changing MoonRay source code, I built an isolated OpenVDB
9.1.0 from tag:

``` text
v9.1.0
```

and installed it to:

``` text
~/Documents/OpenSource/moonray-deps/openvdb-install
```

The successful configuration was:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake ../OpenVDB \
  -DCMAKE_INSTALL_PREFIX=$HOME/Documents/OpenSource/moonray-deps/openvdb-install \
  -DCMAKE_PREFIX_PATH="$HOME/Documents/OpenSource/moonray-deps/openexr-install;$HOME/Documents/OpenSource/moonray-deps/tbb-install" \
  -DImath_DIR=$HOME/Documents/OpenSource/moonray-deps/openexr-install/lib/cmake/Imath \
  -DTbb_INCLUDE_DIR=$HOME/Documents/OpenSource/moonray-deps/tbb-install/include \
  -DTbb_LEGACY_INCLUDE_DIR=$HOME/Documents/OpenSource/moonray-deps/tbb-install/include \
  -DTBB_LIBRARYDIR=$HOME/Documents/OpenSource/moonray-deps/tbb-install/lib \
  -DOPENVDB_BUILD_CORE=ON \
  -DOPENVDB_BUILD_BINARIES=OFF \
  -DOPENVDB_BUILD_PYTHON_MODULE=OFF \
  -DOPENVDB_BUILD_UNITTESTS=OFF \
  -DOPENVDB_BUILD_DOCS=OFF \
  -DOPENVDB_BUILD_HOUDINI_PLUGIN=OFF \
  -DOPENVDB_BUILD_MAYA_PLUGIN=OFF \
  -DOPENVDB_BUILD_AX=OFF \
  -DOPENVDB_BUILD_NANOVDB=OFF \
  -DUSE_TBB=ON \
  -DUSE_BLOSC=ON \
  -DUSE_ZLIB=ON \
  -DUSE_EXR=OFF \
  -DUSE_IMATH_HALF=ON
```

This OpenVDB build used TBB 2020.3, Imath 3.1, Blosc, and zlib.

I verified the TBB dependency using `ldd`; it resolved to `libtbb.so.2`
from the custom TBB installation.

------------------------------------------------------------------------

## 21. Make MoonRay Rediscover OpenVDB

After installing the custom OpenVDB:

``` bash
export OPENVDB_ROOT=$HOME/Documents/OpenSource/moonray-deps/openvdb-install
```

From the MoonRay build directory, I cleared cached OpenVDB variables and
reconfigured:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake \
  -UOpenVDB_INCLUDE_DIR \
  -UOpenVDB_INCLUDE_DIRS \
  -UOpenVDB_LIBRARIES \
  .
```

CMake then selected the custom OpenVDB installation instead of Ubuntu's
system OpenVDB.

This fixed the `PathIntegrator.cc` OpenVDB/Imath conflict.

------------------------------------------------------------------------

## 22. Create the MoonRay Build Directory

MoonRay was built in:

``` text
~/Documents/OpenSource/openmoonray/build
```

I created the build directory with:

``` bash
cd ~/Documents/OpenSource/openmoonray

rm -rf build
mkdir build
cd build
```

This kept generated files separate from tracked source files.

------------------------------------------------------------------------

## 23. MoonRay CMake Configuration

The successful base configuration was:

``` bash
cd ~/Documents/OpenSource/openmoonray/build

export RANDOM123_ROOT=$HOME/Documents/OpenSource/moonray-deps/Random123

~/cmake-3.31.12-linux-x86_64/bin/cmake .. \
  -DCMAKE_PREFIX_PATH="$HOME/Documents/OpenSource/moonray-deps/embree-install;$HOME/Documents/OpenSource/moonray-deps/openimagedenoise-install;$HOME/Documents/OpenSource/moonray-deps/openimageio-install;$HOME/Documents/OpenSource/moonray-deps/openocio-install;$HOME/Documents/OpenSource/moonray-deps/openexr-install;$HOME/Documents/OpenSource/moonray-deps/usd-install;$HOME/Documents/OpenSource/moonray-deps/opensubdiv-install;$HOME/Documents/OpenSource/moonray-deps/tbb-install" \
  -Dpxr_DIR=$HOME/Documents/OpenSource/moonray-deps/usd-install \
  -DOpenImageIO_DIR=$HOME/Documents/OpenSource/moonray-deps/openimageio-install/lib/cmake/OpenImageIO \
  -DOpenImageDenoise_DIR=$HOME/Documents/OpenSource/moonray-deps/openimagedenoise-install/lib/cmake/OpenImageDenoise-2.3.3 \
  -DCMAKE_ISPC_COMPILER=$HOME/Documents/OpenSource/moonray-deps/ispc-v1.21.0-linux/bin/ispc \
  -DPYTHON_EXECUTABLE=/usr/bin/python3 \
  -DBUILD_QT_APPS=NO \
  -DMOONRAY_USE_OPTIX=NO \
  -DABI_VERSION=0
```

CUDA being unavailable was intentional because OptiX was disabled.

Some CMake developer warnings and optional dependency messages were not
blockers.

------------------------------------------------------------------------

## 24. First Build Failure: Lua

At the beginning of the first MoonRay compilation, the build stopped
because the Lua executable/compiler was missing.

I installed Lua 5.4. Afterward the system provided:

``` text
/bin/lua
/bin/luac
```

MoonRay's CMake cache detected those executables.

I reconfigured MoonRay and continued the build.

At one point Ubuntu displayed a crash notification involving `luac5.4` /
a double-free. The overall build continued, so I did not modify MoonRay
source to address it.

------------------------------------------------------------------------

## 25. Build MoonRay

From the build directory:

``` bash
cd ~/Documents/OpenSource/openmoonray/build
```

I built with four parallel jobs:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake --build . -j4
```

I used `-j4` rather than all eight logical CPUs to avoid unnecessarily
stressing the laptop during the large C++ build.

------------------------------------------------------------------------

## 26. OpenVDB/Imath Failure Around 76%

The build progressed to approximately 76% before failing around
`PathIntegrator.cc`.

The problem was the mixed OpenVDB/OpenEXR/Imath dependency environment
described earlier.

The fix was not a MoonRay source modification. I:

1.  Built OpenVDB 9.1.0 separately.
2.  Built it against the compatible Imath/TBB stack.
3.  Set `OPENVDB_ROOT`.
4.  Cleared MoonRay's cached OpenVDB detection.
5.  Reconfigured.
6.  Resumed the build.

After that, MoonRay progressed much farther.

------------------------------------------------------------------------

## 27. Linker Failure Around 96%

The next major failure happened around 96%.

The linker produced a warning involving both:

``` text
libtbb.so.12
libtbb.so.2
```

This was a major clue.

The build also produced unresolved symbols related to
`ImageDistribution`, `ProjectiveCamera`, `PerspectiveCamera`, `Camera`,
and `isRenderCanceled`.

The TBB libraries represented two incompatible generations:

``` text
libtbb.so.12 -> newer oneTBB
libtbb.so.2  -> TBB 2020
```

I investigated the dependency tree instead of changing MoonRay code.

------------------------------------------------------------------------

## 28. Discover OpenImageIO Was Still Using oneTBB

OpenVDB was already correctly using isolated TBB 2020.3, but OpenImageIO
was not.

OIIO had originally detected Ubuntu's TBB 2021.5.

MoonRay was therefore attempting to link a dependency graph containing
both TBB ABIs.

The solution was to rebuild OpenImageIO against TBB 2020.3.

------------------------------------------------------------------------

## 29. OpenImageIO TBB Detection

Inside OIIO's CMake configuration, TBB was discovered with logic
equivalent to:

``` cmake
set (TBB_USE_DEBUG_BUILD OFF)

checked_find_package (TBB 2017
                      SETVARIABLES OIIO_TBB
                      PREFER_CONFIG)
```

Because it preferred a CMake CONFIG package, simply setting a general
TBB root was not sufficient.

Ubuntu exposed system TBB CMake configuration under both:

``` text
/usr/lib/x86_64-linux-gnu/cmake/TBB
/lib/x86_64-linux-gnu/cmake/TBB
```

Both needed to be ignored during OIIO configuration.

------------------------------------------------------------------------

## 30. Clean OpenImageIO Build

I entered:

``` bash
cd ~/Documents/OpenSource/moonray-deps/openimageio-build
```

and cleared the existing build files:

``` bash
rm -rf ./*
```

Then configured OIIO using:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake ../OpenImageIO \
  -DCMAKE_INSTALL_PREFIX=$HOME/Documents/OpenSource/moonray-deps/openimageio-install \
  -DCMAKE_PREFIX_PATH="$HOME/Documents/OpenSource/moonray-deps/openexr-install;$HOME/Documents/OpenSource/moonray-deps/openocio-install" \
  -DCMAKE_IGNORE_PATH="/usr/lib/x86_64-linux-gnu/cmake/TBB;/lib/x86_64-linux-gnu/cmake/TBB" \
  -DOpenEXR_ROOT=$HOME/Documents/OpenSource/moonray-deps/openexr-install \
  -DOpenEXR_DIR=$HOME/Documents/OpenSource/moonray-deps/openexr-install/lib/cmake/OpenEXR \
  -DImath_DIR=$HOME/Documents/OpenSource/moonray-deps/openexr-install/lib/cmake/Imath \
  -DOpenColorIO_DIR=$HOME/Documents/OpenSource/moonray-deps/openocio-install/lib/cmake/OpenColorIO \
  -DTBB_ROOT_DIR=$HOME/Documents/OpenSource/moonray-deps/tbb-install \
  -DTBB_INCLUDE_DIR=$HOME/Documents/OpenSource/moonray-deps/tbb-install/include \
  -DTBB_LIBRARY=$HOME/Documents/OpenSource/moonray-deps/tbb-install/lib \
  -DUSE_OpenVDB=OFF \
  -DBUILD_SHARED_LIBS=ON \
  -DUSE_QT=OFF
```

Important configuration output:

``` text
-- Could NOT find TBB (missing: TBB_DIR)
-- Found TBB 2020.3
```

The first line was not the actual failure it might appear to be. It
meant the system CONFIG-mode package had not been selected. OIIO
subsequently found the desired TBB 2020.3 installation.

------------------------------------------------------------------------

## 31. Verify OIIO's TBB Configuration

The resulting CMake cache contained the custom TBB paths, including:

``` text
TBB_DIR:PATH=TBB_DIR-NOTFOUND
TBB_INCLUDE_DIRS:PATH=$HOME/Documents/OpenSource/moonray-deps/tbb-install/include
TBB_tbb_LIBRARY_RELEASE:FILEPATH=$HOME/Documents/OpenSource/moonray-deps/tbb-install/lib/libtbb.so
TBB_tbbmalloc_LIBRARY_RELEASE:FILEPATH=$HOME/Documents/OpenSource/moonray-deps/tbb-install/lib/libtbbmalloc.so
```

I then built OIIO and installed it:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake --install .
```

------------------------------------------------------------------------

## 32. Verify OIIO Is Actually Using TBB 2020

I ran:

``` bash
ldd ~/Documents/OpenSource/moonray-deps/openimageio-install/lib/libOpenImageIO_Util.so.2.4.8 | grep -i tbb
```

The result pointed to:

``` text
libtbb.so.2
```

under:

``` text
~/Documents/OpenSource/moonray-deps/tbb-install/lib/
```

This confirmed OIIO was no longer pulling in Ubuntu's `libtbb.so.12`.

------------------------------------------------------------------------

## 33. Resume MoonRay

I returned to:

``` bash
cd ~/Documents/OpenSource/openmoonray/build
```

and rebuilt.

The previous TBB warning disappeared. The unresolved
Camera/ImageDistribution-related symbols also disappeared.

Only one major linker error remained:

``` text
undefined reference to `isRenderCanceled'
```

from `librendering_pbr.so` while linking `moonray_rendering_pbr_tests`.

------------------------------------------------------------------------

## 34. Investigate `isRenderCanceled`

Rather than adding a fake implementation, I searched MoonRay source:

``` bash
grep -Rni --exclude-dir=build \
  'isRenderCanceled' \
  ~/Documents/OpenSource/openmoonray/moonray \
  | head -100
```

The search showed references including:

``` text
moonray/lib/rendering/rndr/RenderDriver.h
moonray/lib/rendering/rndr/RenderDriver.cc
moonray/lib/rendering/rndr/RenderFramePasses.cc
moonray/lib/rendering/pbr/core/PbrTLState.ispc
moonray/lib/rendering/pbr/core/PbrTLState.cc
```

`RenderDriver.cc` contained the C-linkage implementation:

``` cpp
extern "C" bool isRenderCanceled()
```

That file belongs to `rendering_rndr`, while PBR was referencing the
function.

------------------------------------------------------------------------

## 35. Verify the Symbol Exists

I checked the built rendering library directly:

``` bash
nm -D \
  ~/Documents/OpenSource/openmoonray/build/moonray/moonray/lib/rendering/rndr/librendering_rndr.so \
  | grep isRenderCanceled
```

It returned:

``` text
0000000000220ca0 T isRenderCanceled
```

The function was therefore not missing. It existed and was exported by
`librendering_rndr.so`.

Adding or changing source code would have been the wrong fix.

------------------------------------------------------------------------

## 36. Inspect the Link Order

The PBR test CMake configuration already linked both:

``` text
Moonray::rendering_pbr
Moonray::rendering_rndr
```

However, inspection of the verbose linker command showed that
`rendering_rndr.so` appeared before `rendering_pbr.so` in the final
executable's link command.

On this Ubuntu environment, linker `--as-needed` behavior could discard
a shared library when it appeared before the library that introduced the
unresolved symbol requiring it.

This explained why `rendering_rndr` contained `isRenderCanceled`, yet
the executable still reported the function as undefined.

------------------------------------------------------------------------

## 37. Test `--no-as-needed`

Instead of changing MoonRay's CMake source files, I tested the linker
hypothesis by reconfiguring the existing build directory with:

``` bash
cd ~/Documents/OpenSource/openmoonray/build

~/cmake-3.31.12-linux-x86_64/bin/cmake \
  -DCMAKE_EXE_LINKER_FLAGS="-Wl,--no-as-needed" \
  .
```

This is an Ubuntu build workaround, not a claimed upstream MoonRay fix.

------------------------------------------------------------------------

## 38. Build the Previously Failing PBR Test Target

I explicitly built the target that had been failing:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake --build . \
  --target moonray_rendering_pbr_tests \
  -j4
```

This time it completed with:

``` text
[100%] Built target moonray_rendering_pbr_tests
```

That supported the conclusion that the `isRenderCanceled` problem was
related to executable linker behavior/order rather than a missing
MoonRay implementation.

------------------------------------------------------------------------

## 39. Full MoonRay Build

I then built the complete project again:

``` bash
cd ~/Documents/OpenSource/openmoonray/build

~/cmake-3.31.12-linux-x86_64/bin/cmake --build . -j4
```

This time the build completed successfully.

Final output included:

``` text
[100%] Built target HairLayerMaterial
[100%] Built target DwaToonMaterial
```

Important targets also built successfully, including:

``` text
rendering_shading
rendering_pbr
rendering_rndr
moonray_rendering_pbr_tests
moonray
```

There were no compiler or linker errors at the end.

------------------------------------------------------------------------

## 40. Test-Suite Caveat

During testing, I also encountered a large number of failures associated
with OpenImageIO's test setup and missing external test-image data.

For example, an `igrep` test expected:

``` text
../oiio-images/tahoe-gps.jpg
```

but the external image directory was not present:

``` bash
ls -ld ~/Documents/OpenSource/moonray-deps/OpenImageIO/testsuite/oiio-images
```

returned:

``` text
No such file or directory
```

There was also a separate CMake-consumer issue.

For that reason, this guide does **not** claim that every available test
in the complete dependency/MoonRay environment passes. The verified
milestone is that the complete OpenMoonRay build succeeds and the
resulting MoonRay executable starts successfully.

------------------------------------------------------------------------

## 41. Locate the MoonRay Executable

After the successful full build:

``` bash
cd ~/Documents/OpenSource/openmoonray/build
```

I searched for the executable:

``` bash
find . -type f -name moonray -executable
```

Result:

``` text
./moonray/moonray/cmd/raas_cmd/moonray/moonray
```

------------------------------------------------------------------------

## 42. Launch MoonRay

I ran:

``` bash
./moonray/moonray/cmd/raas_cmd/moonray/moonray --help
```

MoonRay successfully started and printed its command-line help.

This demonstrated that the executable was not merely compiled and
linked: it could start and resolve its runtime dependencies.

Useful options exposed by the executable include execution modes:

``` text
scalar
vectorized
xpu
auto
```

and a BSDF debugging option:

``` text
-print_bsdf x y
```

which prints the BSDF configuration for a material during rendering.

------------------------------------------------------------------------

## 43. Verify That MoonRay Source Was Untouched

Finally:

``` bash
cd ~/Documents/OpenSource/openmoonray
git status
```

Result:

``` text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

This confirmed that none of the Ubuntu compatibility work modified
tracked MoonRay source code.

------------------------------------------------------------------------

## 44. What Was Actually Changed?

The process affected three general areas.

### Dependencies

Custom dependencies were built under:

``` text
~/Documents/OpenSource/moonray-deps/
```

### Generated MoonRay Build Files

CMake configuration, compiled libraries, executables, and generated
files were placed under:

``` text
~/Documents/OpenSource/openmoonray/build/
```

### System Packages

Required Ubuntu development packages such as Lua and various development
headers/tools were installed through the package manager.

The tracked MoonRay source on `main` remained unchanged.

------------------------------------------------------------------------

## 45. Major Problems and Solutions

  --------------------------------------------------------------------------
  Problem                    Cause                   Solution
  -------------------------- ----------------------- -----------------------
  System CMake inadequate    Ubuntu 22.04 CMake      Used standalone CMake
                             3.22.1                  3.31.12

  Dependency discovery       Conda environment       Deactivated Conda and
  contaminated                                       explicitly used
                                                     `/usr/bin/python3`

  `pxrConfig.cmake` missing  USD unavailable         Built USD 23.08

  Lua failure near start     Lua executable/compiler Installed Lua 5.4
                             missing                 

  `PathIntegrator.cc` /      System OpenVDB 8.1      Built OpenVDB 9.1.0
  `half` conflict            mixed with custom       against compatible
                             Imath/OpenEXR           dependency stack

  TBB linker warning         OIIO using oneTBB while Rebuilt OIIO against
                             other dependencies used isolated TBB 2020.3
                             TBB 2020                

  Camera/ImageDistribution   Mixed dependency/ABI    Disappeared after
  unresolved symbols         graph                   OIIO/TBB correction

  `isRenderCanceled`         Linker `--as-needed`    Configured executable
  unresolved                 behavior/link ordering  linker with
                                                     `-Wl,--no-as-needed`

  Large OIIO test failure    Missing external test   Diagnosed separately;
  set                        assets plus at least    not treated as a
                             one separate            MoonRay compile failure
                             test/configuration      
                             issue                   

  Concern source had changed Many build workarounds  `git status` confirmed
                             were required           a clean source tree
  --------------------------------------------------------------------------

------------------------------------------------------------------------

## 46. Important Caveat About `--no-as-needed`

The command:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake \
  -DCMAKE_EXE_LINKER_FLAGS="-Wl,--no-as-needed" \
  .
```

was the workaround that allowed this Ubuntu build to link successfully.

I do **not** describe this as the correct upstream fix for OpenMoonRay.

A more precise description is:

> On this Ubuntu 22.04 configuration, `moonray_rendering_pbr_tests`
> failed to resolve `isRenderCanceled` even though the symbol was
> exported by `librendering_rndr.so`. Inspection of the link command
> indicated a shared-library ordering/`--as-needed` interaction.
> Configuring executable linking with `-Wl,--no-as-needed` allowed the
> target and complete project to link successfully.

------------------------------------------------------------------------

## 47. Additional Dependency Observation

During investigation, one verbose link command showed a mixture of
custom Imath 3.1 and some Ubuntu OpenEXR/Imath 2.5 libraries.

The final MoonRay build nevertheless succeeded, so I did not begin
changing otherwise working dependencies merely to make the link command
cleaner.

This remains a potential area for future Ubuntu build cleanup rather
than a demonstrated blocker.

------------------------------------------------------------------------

## 48. Rebuilding From the Working Configuration

Once all dependencies are installed and the build directory has been
configured correctly:

``` bash
cd ~/Documents/OpenSource/openmoonray/build

~/cmake-3.31.12-linux-x86_64/bin/cmake --build . -j4
```

The PBR test executable can be rebuilt independently:

``` bash
~/cmake-3.31.12-linux-x86_64/bin/cmake --build . \
  --target moonray_rendering_pbr_tests \
  -j4
```

MoonRay can be checked with:

``` bash
./moonray/moonray/cmd/raas_cmd/moonray/moonray --help
```

------------------------------------------------------------------------

## 49. Environment Variables to Remember

For a fresh terminal:

``` bash
export RANDOM123_ROOT=$HOME/Documents/OpenSource/moonray-deps/Random123
```

For OpenVDB rediscovery/configuration:

``` bash
export OPENVDB_ROOT=$HOME/Documents/OpenSource/moonray-deps/openvdb-install
```

I intentionally did not permanently add a large collection of custom
library paths to `.bashrc`. This reduces the chance that the MoonRay
development environment interferes with unrelated software.

------------------------------------------------------------------------

## 50. Things I Deliberately Did Not Do

I did not:

-   Modify MoonRay C++ source to make Ubuntu compile.
-   Replace `isRenderCanceled` with my own implementation.
-   Modify PBR source to work around the linker.
-   Permanently replace Ubuntu's system CMake.
-   Rely on Conda's CMake/Python environment.
-   Enable OptiX/CUDA for this first build.
-   Enable the Qt applications.
-   Use AVX-512 on a CPU that does not support it.
-   Assume system OpenVDB was compatible after the Imath failure.
-   Treat dependency test failures as proof that MoonRay itself failed
    to build.

The goal was to establish a clean build baseline before beginning an
actual OpenMoonRay source contribution.

------------------------------------------------------------------------

## 51. Current Status

The resulting environment successfully produced:

``` text
OpenMoonRay source
        ↓
CMake configuration
        ↓
C++ compilation
        ↓
ISPC compilation
        ↓
MoonRay libraries
        ↓
MoonRay shaders/materials
        ↓
moonray executable
        ↓
successful runtime startup
```

And:

``` bash
git status
```

still reports:

``` text
nothing to commit, working tree clean
```

------------------------------------------------------------------------

## Disclaimer

This is an unofficial community guide documenting the environment and
steps I used to successfully build OpenMoonRay on Ubuntu 22.04.5 LTS.
OpenMoonRay's tested Linux build environment is Rocky Linux 9. These
instructions should not be interpreted as an officially supported Ubuntu
configuration. Some dependency or linker adjustments may be specific to
this system.

## Notes on Reproducibility

This README records the build and troubleshooting process that was
actually used. Some early dependency compilation commands were not
preserved in full, so they have not been reconstructed from guesswork.
Where exact commands were retained---particularly the MoonRay
configuration, OpenVDB configuration, OpenImageIO/TBB correction, linker
workaround, and final verification---they are included directly above.
