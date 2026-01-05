How do I detect OpenMP support in an R package?
===============================================

It depends on the target operating system
-----------------------------------------

### Most operating systems, including Linux and Windows

In your `Makevars`, set the `PKG_CFLAGS` and `PKG_LIBS` using the Make
macros described in [Writing R Extensions][WRE-OpenMP]. If the OpenMP
support was detected and enabled during R configuration, your package
will use a compatible OpenMP runtime. If not (e.g. disabled by system
administrator), OpenMP support will be disabled, so make sure to use
`#ifdef _OPENMP` (or other tests appropriate for your language).

### macOS

OpenMP isn't really supported by the Apple toolchain, but [with clever
hacks][mac-openmp] you can get it to work. R does not detect OpenMP
support by default, so the Make macros described above will be empty.

Nevertheless, starting with version 4.3, R for macOS provides an OpenMP
runtime and a way for packages to [opt into][SIG-Mac-Apr25] using it on
the CRAN package builder:

> the main part is to detect OpenMP even if R doesn't enable it
> (possibly as an option?) and on macOS also try `-Xclang` in front of
> the regular `-fopenmp` to see if it works

(In addition to testing for `-Xclang -fopenmp`, the package also needs
to link to the OpenMP runtime using the `-lomp` linker flag.)

This package demonstrates how to achieve that. It is important to test
multiple configurations on macOS to make sure that installing from
source in weird cases (e.g. custom toolchain that understands
`-fopenmp`) will still work.

As of 2025, newer Xcode versions conflict with the OpenMP runtime
shipped with R. Use `schedule(dynamic)` to [check for it][datatable7318]
and avoid accidentally building a package shared library that will later
fail to load. Installing a [different version of the OpenMP
runtime][mac-openmp] breaks compatibility with CRAN binaries, so it's up
to the administrator to set `SHLIB_OPENMP_CFLAGS = -Xclang -fopenmp
-Wl,/usr/local/lib/libomp.dylib` (untested) and compile all packages
from source.

Contents
--------

### `src/Makevars.win`

Adds `$(SHLIB_OPENMP_CFLAGS)` to the compiler and linker flags on
Windows. R will warn about the package having a `configure` script for
Unix-alikes but not for Windows, but we already have `src/Makevars.win`
pre-configured.

See [WRE 1.1.5 Package subdirectories][WRE-package-subdirectories] for
more information.

### `configure`

On Unix-alikes, [`configure`][WRE-configure] is required to be a POSIX
shell script. In theory, it could [immediately delegate to an R
script][Kevin-Ushey-configure], but we will only use it for
compile-testing; the rest of the script is more laconic when written in
POSIX `sh`.

A good `configure` leaves a `config.log` with detailed information for
debugging. POSIX `echo` is not guaranteed to understand `-n` or escape
characters, so we'll use `printf` instead.

On macOS (whose `uname` calls it "Darwin"), we call the compile-test
function with different arguments, until one succeeds, or until we reach
the last case, which leaves the OpenMP variables empty. On other
operating systems, we rely on R-provided flags unconditionally.

The compile-test function runs R in order to `dyn.load()` the resulting
shared library and run code from it. It tests the OpenMP support as
completely as it can:

1. Call `R CMD SHLIB` to compile and link a shared library from the test
   C file.
2. Load the resulting shared library and run an OpenMP loop from it.

This aims to trigger all of compilation-time, linking-time,
loading-time, and runtime problems before they prevent the real package
from being installed.

The last step is the [autotools]-style text replacement that takes the
`src/Makevars.in` file and creates the `src/Makevars` from it for
`R CMD SHLIB` to consume.

### `src/Makevars.in`

This prototype for `src/Makevars` contains templates for compiler and
linker flags that the `configure` script will substitute.

### `cleanup`

[For best results][WRE-configure], `configure` should be paired with a
`cleanup` script, which removes all files that `configure` may have
created, although it's prudent to also list them in `.Rbuildignore`. In
our case, these files are `src/Makevars` and `config.log`.

Bonus points: passing through `PKG_CFLAGS`, `PKG_LIBS`
------------------------------------------------------

The [OpenMP on macOS][mac-openmp] page recommends:

> How you do the latter depends on the package, but if the package does not set
> these environment variables itself, you can try
>
>     PKG_CPPFLAGS='-Xclang -fopenmp' PKG_LIBS=-lomp R CMD INSTALL myPackage

Since our `Makevars` file sets `PKG_CPPFLAGS` and `PKG_LIBS`, Make won't
read them from the environment. Solution: read them in the `configure`
script and write them into `Makevars`, in addition to other variables
being set, using the same string substitution as the OpenMP flags. For
maximum compatibility, test a configuration with no added compiler or
linker flags, making it possible for the user to override OpenMP
configuration.

Results
-------

### macOS 26.2, R-devel, Apple clang 17.0.0 ([macOS builder][mac-builder], R-devel)

```
* installing *source* package ‘ompdetect’ ...
** this is package ‘ompdetect’ version ‘0.2-0’
** using staged installation
Variables from the environment: PKG_CFLAGS='' PKG_LIBS=''
Guessing OpenMP configuration for macOS
* checking if OpenMP works with PKG_CFLAGS='' PKG_LIBS=''... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp'... yes
Adding PKG_CFLAGS='-Xclang -fopenmp', PKG_LIBS='-lomp'
** libs
using C compiler: ‘Apple clang version 17.0.0 (clang-1700.6.3.2)’
using SDK: ‘MacOSX14.4.sdk’
clang -arch arm64 -std=gnu23 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I/opt/R/arm64/include   -Xclang -fopenmp -fPIC  -falign-functions=64 -Wall -g -O2  -c test_omp.c -o test_omp.o
clang -arch arm64 -std=gnu23 -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -L/Library/Frameworks/R.framework/Resources/lib -L/opt/R/arm64/lib -o ompdetect.so test_omp.o -lomp -F/Library/Frameworks/R.framework/.. -framework R
installing to /Volumes/PkgBuild/work/1767639495-8aa9b6af8ed5d0a9/packages/sonoma-arm64/results/4.6/ompdetect.Rcheck/00LOCK-ompdetect/00new/ompdetect/libs
```

```
> ### ** Examples
>
>   ompdetect()
OpenMP detected and working.
>   omplimits()
thread_limit  max_threads    num_procs
  2147483647            8            8
```

### macOS 15.7.2, R-4.5.2, Apple clang 17.0.0 (GitHub Actions)

The compiler is too new for the R-bundled OpenMP runtime, and the resulting
shared object compiles and links successfully, but fails to load. This can
happen in GitHub Actions without extra preparations.

```
* installing *source* package ‘ompdetect’ ...
** this is package ‘ompdetect’ version ‘0.2-0’
** using staged installation
Variables from the environment: PKG_CFLAGS='' PKG_LIBS=''
Guessing OpenMP configuration for macOS
* checking if OpenMP works with PKG_CFLAGS='' PKG_LIBS=''... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp'... failed to build
* checking if OpenMP works with PKG_CFLAGS='$(SHLIB_OPENMP_CFLAGS)' PKG_LIBS='$(SHLIB_OPENMP_CFLAGS)'... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='/usr/local/lib/libomp.dylib'... failed to build
Couldn't guess a working OpenMP configuration.
Do you need an OpenMP runtime and headers from <https://mac.r-project.org/openmp/>?
Adding PKG_CFLAGS='', PKG_LIBS=''
** libs
using C compiler: ‘Apple clang version 17.0.0 (clang-1700.0.13.5)’
using SDK: ‘MacOSX15.5.sdk’
clang -arch arm64 -std=gnu2x -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I/opt/R/arm64/include    -fPIC  -falign-functions=64 -Wall -g -O2  -c test_omp.c -o test_omp.o
clang -arch arm64 -std=gnu2x -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -L/Library/Frameworks/R.framework/Resources/lib -L/opt/R/arm64/lib -o ompdetect.so test_omp.o -F/Library/Frameworks/R.framework/.. -framework R
installing to /Users/runner/work/_temp/Library/00LOCK-ompdetect/00new/ompdetect/libs
```

```
$ Rscript -e 'ompdetect::ompdetect(); ompdetect::omplimits()'
OpenMP not detected.
thread_limit  max_threads    num_procs
          -1           -1           -1
```

### macOS 15.7.2, R-4.5.2, Apple clang 17.0.0, compatible OpenMP runtime in `/usr/local`

Same as above, but with a [compatible OpenMP runtime][mac-openmp] installed:

```
* installing *source* package ‘ompdetect’ ...
** this is package ‘ompdetect’ version ‘0.2-0’
** using staged installation
Variables from the environment: PKG_CFLAGS='' PKG_LIBS=''
Guessing OpenMP configuration for macOS
* checking if OpenMP works with PKG_CFLAGS='' PKG_LIBS=''... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp'... failed to run
* checking if OpenMP works with PKG_CFLAGS='$(SHLIB_OPENMP_CFLAGS)' PKG_LIBS='$(SHLIB_OPENMP_CFLAGS)'... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='/usr/local/lib/libomp.dylib'... yes
Since an earlier test for -lomp failed, the resulting package is probably unsafe to mix with CRAN binaries!
Adding PKG_CFLAGS='-Xclang -fopenmp', PKG_LIBS='/usr/local/lib/libomp.dylib'
** libs
using C compiler: ‘Apple clang version 17.0.0 (clang-1700.0.13.5)’
using SDK: ‘MacOSX15.5.sdk’
clang -arch arm64 -std=gnu2x -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I/opt/R/arm64/include   -Xclang -fopenmp -fPIC  -falign-functions=64 -Wall -g -O2  -c test_omp.c -o test_omp.o
clang -arch arm64 -std=gnu2x -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -L/Library/Frameworks/R.framework/Resources/lib -L/opt/R/arm64/lib -o ompdetect.so test_omp.o /usr/local/lib/libomp.dylib -F/Library/Frameworks/R.framework/.. -framework R
installing to /Users/runner/work/_temp/Library/00LOCK-ompdetect/00new/ompdetect/libs
```

```
$ Rscript -e 'ompdetect::ompdetect(); ompdetect::omplimits()'
OpenMP detected and working.
thread_limit  max_threads    num_procs
  2147483647            3            3
```


### macOS 15.7.2, R-4.5.2, Apple clang version 16.0.0, OpenMP headers installed

Switching Xcode version to 16.2 makes it possible to use a compiler
version compatible with the OpenMP runtime used by R-4.5.2.
[Installing][mac-openmp] the `omp.h` header is still necessary as it's
not bundled with R.

```
* installing *source* package ‘ompdetect’ ...
** this is package ‘ompdetect’ version ‘0.2-0’
** using staged installation
Variables from the environment: PKG_CFLAGS='' PKG_LIBS=''
Guessing OpenMP configuration for macOS
* checking if OpenMP works with PKG_CFLAGS='' PKG_LIBS=''... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp'... yes
Adding PKG_CFLAGS='-Xclang -fopenmp', PKG_LIBS='-lomp'
** libs
using C compiler: ‘Apple clang version 16.0.0 (clang-1600.0.26.6)’
using SDK: ‘MacOSX15.2.sdk’
clang -arch arm64 -std=gnu2x -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I/opt/R/arm64/include   -Xclang -fopenmp -fPIC  -falign-functions=64 -Wall -g -O2  -c test_omp.c -o test_omp.o
clang -arch arm64 -std=gnu2x -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -L/Library/Frameworks/R.framework/Resources/lib -L/opt/R/arm64/lib -o ompdetect.so test_omp.o -lomp -F/Library/Frameworks/R.framework/.. -framework R
installing to /Library/Frameworks/R.framework/Versions/4.5-arm64/Resources/library/00LOCK-ompdetect/00new/ompdetect/libs
```

```
$ Rscript -e 'ompdetect::ompdetect(); ompdetect::omplimits()'
OpenMP detected and working.
thread_limit  max_threads    num_procs
  2147483647            3            3
```

### macOS 10.15.7, R-4.2.3, Apple clang 12.0.0, compatible OpenMP runtime in `/usr/local`

Since the bundled OpenMP runtime only appeared in R-4.3, this
necessitated a manual installation of the [OpenMP runtime][mac-openmp]
(`openmp-10.0.0-darwin17-Release.tar.gz`).

```
* installing *source* package ‘ompdetect’ ...
** using staged installation
Variables from the environment: PKG_CFLAGS='' PKG_LIBS=''
Guessing OpenMP configuration for macOS
* checking if OpenMP works with PKG_CFLAGS='' PKG_LIBS=''... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp'... yes
Adding PKG_CFLAGS='-Xclang -fopenmp', PKG_LIBS='-lomp'
** libs
clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I/usr/local/include  -Xclang -fopenmp -fPIC  -Wall -g -O2  -c test_omp.c -o test_omp.o
clang -mmacosx-version-min=10.13 -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -single_module -multiply_defined suppress -L/Library/Frameworks/R.framework/Resources/lib -L/usr/local/lib -o ompdetect.so test_omp.o -lomp -F/Library/Frameworks/R.framework/.. -framework R -Wl,-framework -Wl,CoreFoundation
installing to /Library/Frameworks/R.framework/Versions/4.2/Resources/library/00LOCK-ompdetect/00new/ompdetect/libs
```
```
$ Rscript -e 'ompdetect::ompdetect(); ompdetect::omplimits()'
OpenMP detected and working.
thread_limit  max_threads    num_procs
  2147483647            3            3
```

### Flags specified in the environment

Same as above, but the `configure` script takes the flags manually
specified in the environment variables.

```
$ PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp' R CMD INSTALL ompdetect_0.2-0.tar.gz
* installing to library ‘/Library/Frameworks/R.framework/Versions/4.2/Resources/library’
* installing *source* package ‘ompdetect’ ...
** using staged installation
Variables from the environment: PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp'
Guessing OpenMP configuration for macOS
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp ' PKG_LIBS='-lomp '... yes
Adding PKG_CFLAGS='-Xclang -fopenmp ', PKG_LIBS='-lomp '
** libs
clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I/usr/local/include  -Xclang -fopenmp  -fPIC  -Wall -g -O2  -c test_omp.c -o test_omp.o
clang -mmacosx-version-min=10.13 -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -single_module -multiply_defined suppress -L/Library/Frameworks/R.framework/Resources/lib -L/usr/local/lib -o ompdetect.so test_omp.o -lomp -F/Library/Frameworks/R.framework/.. -framework R -Wl,-framework -Wl,CoreFoundation
installing to /Library/Frameworks/R.framework/Versions/4.2/Resources/library/00LOCK-ompdetect/00new/ompdetect/libs
```

### macOS 10.15.7, R-4.5.2 from MacPorts, clang 17.0.6

The toolchain used by MacPorts supports OpenMP natively, so R detects
OpenMP support and provides the necessary flags (`-fopenmp`) in the
`$(SHLIB_OPENMP_CFLAGS)` Make macro:


```
* installing *source* package ‘ompdetect’ ...
** this is package ‘ompdetect’ version ‘0.2-0’
** using staged installation
Variables from the environment: PKG_CFLAGS='' PKG_LIBS=''
Guessing OpenMP configuration for macOS
* checking if OpenMP works with PKG_CFLAGS='' PKG_LIBS=''... failed to run
* checking if OpenMP works with PKG_CFLAGS='-Xclang -fopenmp' PKG_LIBS='-lomp'... failed to build
* checking if OpenMP works with PKG_CFLAGS='$(SHLIB_OPENMP_CFLAGS)' PKG_LIBS='$(SHLIB_OPENMP_CFLAGS)'... yes
Adding PKG_CFLAGS='$(SHLIB_OPENMP_CFLAGS)', PKG_LIBS='$(SHLIB_OPENMP_CFLAGS)'
** libs
using C compiler: ‘clang version 17.0.6’
using SDK: ‘MacOSX10.15.6.sdk’
/opt/local/bin/clang-mp-17 -std=gnu2x -I"/opt/local/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I/opt/local/include -isysroot/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk   -fopenmp -fPIC  -pipe -Os -isysroot/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk -arch x86_64  -c test_omp.c -o test_omp.o
/opt/local/bin/clang-mp-17 -std=gnu2x -dynamiclib -Wl,-headerpad_max_install_names -undefined dynamic_lookup -single_module -multiply_defined suppress -L/opt/local/Library/Frameworks/R.framework/Resources/lib -L/opt/local/lib -Wl,-headerpad_max_install_names -Wl,-rpath,/opt/local/lib/libgcc -Wl,-syslibroot,/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk -arch x86_64 -o ompdetect.so test_omp.o -fopenmp -F/opt/local/Library/Frameworks/R.framework/.. -framework R
installing to /Users/user/Library/R/x86_64/4.5/library/00LOCK-ompdetect/00new/ompdetect/libs
```

This should work in a similar manner with a Homebrew build of R.

[WRE-OpenMP]: https://cran.r-project.org/doc/manuals/R-exts.html#OpenMP-support
[mac-openmp]: https://mac.r-project.org/openmp/
[SIG-Mac-Apr25]: https://stat.ethz.ch/pipermail/r-sig-mac/2025-April/015189.html
[datatable7318]: https://github.com/Rdatatable/data.table/pull/7318#issuecomment-3311861544
[WRE-package-subdirectories]: https://cran.r-project.org/doc/manuals/R-exts.html#Package-subdirectories
[WRE-configure]: https://cran.r-project.org/doc/manuals/R-exts.html#Configure-and-cleanup
[Kevin-Ushey-configure]: https://github.com/kevinushey/configure
[autotools]: https://autotools.info/
[mac-builder]: https://mac.r-project.org/macbuilder/submit.html
