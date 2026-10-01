# AGENTS.md

## Cursor Cloud specific instructions

HDF5 is a C/C++/Fortran/Java data-storage library built with CMake. The "application" is
the library plus its command-line tools (`h5dump`, `h5ls`, `h5stat`, `h5repack`, etc.).
General build docs live in `docs/INSTALL_CMake.md`; contributor info in `CONTRIBUTING.md`.

The dependency toolchain (compilers, ninja, zlib, libaec/szip, clang-format-17, codespell)
is installed by the startup update script, so it is already present in this environment.

### Build / run / test (out-of-source build in `build/`, which is git-ignored)

Key gotcha: the default `/usr/bin/c++` and `/usr/bin/cc` resolve through
`/etc/alternatives` to **clang**, and clang here cannot find `libstdc++` (link error
`cannot find -lstdc++`). Always force the GNU compilers when configuring, e.g.:

```bash
cmake -S . -B build -G Ninja \
  -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ -DCMAKE_Fortran_COMPILER=gfortran \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_SHARED_LIBS=ON -DBUILD_TESTING=ON \
  -DHDF5_BUILD_CPP_LIB=ON -DHDF5_BUILD_FORTRAN=ON -DHDF5_BUILD_JAVA=ON \
  -DHDF5_BUILD_HL_LIB=ON -DHDF5_BUILD_TOOLS=ON -DHDF5_BUILD_EXAMPLES=ON \
  -DHDF5_ENABLE_ZLIB_SUPPORT=ON -DHDF5_ENABLE_SZIP_SUPPORT=ON -DHDF5_ENABLE_SZIP_ENCODING=ON
cmake --build build -j"$(nproc)"      # full build is ~4200 targets
ctest --test-dir build -j"$(nproc)"   # 3498 tests; run subsets with -R, e.g. -R H5TEST-testhdf5
```

Notes:
- This command-mode build uses the **system** zlib and libaec (szip). The
  `cmake --preset ci-StdShar-GNUC` workflow presets instead download zlib/libaec from
  GitHub at configure time and need network access.
- Many ctest entries are multi-step pipelines (a `*-clear-objects` / create step feeds a
  later compare step). Running only a sub-step via `-R` can spuriously "fail" because its
  prerequisite did not run; run the whole group (e.g. `-R H5MKGRP-h5mkgrp_single`) instead.
- When compiling your own C program against the in-tree build, include **both** `src` and
  `src/H5FDsubfiling`, e.g.
  `gcc app.c -I src -I src/H5FDsubfiling -I build/src -L build/bin -lhdf5 -Wl,-rpath,build/bin`.
- Built tools and shared libraries land in `build/bin/`.

### Lint

CI uses clang-format **v17** and codespell (configs: `.clang-format`, `.codespellrc`):

```bash
clang-format-17 --style=file --dry-run --Werror <file.c>
codespell <paths>
```
