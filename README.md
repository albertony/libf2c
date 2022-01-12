# About

This repository contains the source code and a Visual Studio 2015  / Visual C++ 14.0
project (easily upgradable to newer versions) for building a static
library containing the necessary runtime components for programs converted
from Fortran77 to C using the fortran77-to-c source code translator **f2c**.

The f2c utility is based on the original Fortran77 compiler **f77**,
and therefore the C code generated by it must be linked against it's
runtime components **libf77** and **libi77**.
Originally these were two separate libraries, but they were at some point combined
into a more convenient single library, **libf2c**.

See also my [f2c](https://github.com/albertony/f2c) project, for building
the source code translator program.

The f2c utility is still actively maintained and is available at
<http://www.netlib.org/f2c/>. Changelog can be found at <http://www.netlib.org/f2c/changes>.
The source code of the combined runtime library is available as
[libf2c.zip](http://www.netlib.org/f2c/libf2c.zip).
Read more in <http://www.netlib.org/f2c/README>.

The contribution of this repository is the Visual Studio project, the source
code is the original libf2c source code available at netlib.org.

## Changelog

See <http://www.netlib.org/f2c/changes> for history of changes in the netlib f2c source,
and if you have the translator program run `f2c.exe --version` to see which version it
was built from.

12 Jan 2022
- Updated to latest version of netlib f2c source, version 20210928
  (but it did not include any relevant changes for libf2c).
- Released as version 1.1, with release builds of libf2c.lib in 32-bit
and 64-bit using Visual Studio 2022 / Visual C++ 14.3.

22 Mar 2017
- Initial version, based on netlib f2c source version 20100827.
- Released as version 1.0, with release builds of libf2c.lib in 32-bit
and 64-bit using Visual Studio 2015 Update 3 / Visual C++ 14.0.

# How to use

The repository includes the original unmodified source code of the libf2c utility
as provided by netlib.org, so all you have to do is use Visual Studio to
build the supplied project **libf2c.vcxproj**. This produces the static library
**libf2c.lib**, which contains the runtime components you need to link together
with your Fortran-to-C converted application. You could typically add the libf2c
project to the solution where you have your Fortran-to-C converted projects,
and specify a project reference from those projects to libf2c.

If you want to use your own copy of the libf2c source code you just have to
get the project file from this repository, and then download the latest version
of libf2c.zip from <http://www.netlib.org/f2c/>
(direct link: <http://www.netlib.org/f2c/libf2c.zip>).
Extract the contents to a subdirectory with name "src". The final step is to
rename four files: Change the file extension of all .h0 and .hvc files to .h.
This includes the files: sysdep1.h0, signal1.h0, math.hvc and f2c.h0. You can
use the following PowerShell snippet to rename all at once:

```powershell
Get-ChildItem *.h0, *.hvc | ForEach-Object { Rename-Item $_ -NewName "$($_.BaseName).h" }
```

# How the Visual Studio project was created

The following steps describe how the Visual Studio project was created, so by
following them you can do it all yourself without using any files from this repository.
The steps are based on the VC makefile (makefile.vc) included with the source code,
and then a substantial portion of my own trial and error.

1. Prepare the source code:
   1. Download latest version of libf2c.zip from <http://www.netlib.org/f2c/>
      (direct link: <http://www.netlib.org/f2c/libf2c.zip>).
   2. Extract libf2c.zip to subdirectory "src".
   3. Change file extension of any *.h0 and *.hvc to *.h. This includes
      sysdep1.h0, signal1.h0, math.hvc and f2c.h0. Use the following
      PowerShell command to rename all at once:

      ```powershell
      Get-ChildItem *.h0, *.hvc | ForEach-Object { Rename-Item $_ -NewName "$($_.BaseName).h" }
      ```

2. Create a new project "libf2c" of type static library, without precompiled header.

3. Add all source files, extension .c, to the project, except the following:
   * ctype.c
   * ftell64_.c
   * pow_qq.c
   * qbitbits.c
   * qbitshft.c
   * signbit.c

4. Add all header files, files with extension .h (including those previously
   renamed from .h0 and .hvc), to the project, except the following:
   * rawio.h

5. Set up generation of arith.h: The compilation of libf2c is designed such that 
   it first needs to build a command line utility **arithchk.exe**, which then is executed
   to check the current system, and the information is stored in a header file (arith.h). 
   The content will typically be something like the following when compiling for x86:

   ```c
   #define IEEE_8087
   #define Arith_Kind_ASL 1
   #define Double_Align
   #define ssize_t long
   #define QNaN0 0x0
   #define QNaN1 0xfff80000
   ```

   and something linke this when comiling for x64:

   ```c
   #define IEEE_8087
   #define Arith_Kind_ASL 1
   #define Double_Align
   #define X64_bit_pointers
   #define LONG_LONG_POINTERS
   #define ssize_t long long
   #define QNaN0 0x0
   #define QNaN1 0xfff80000
   ```

   The header file arith.h is included by the source file uninit.c, so it must
   be generated before this file is being compiled. One (maybe a bit hacky) way
   of doing this is to create a Pre-Build Event on the project which does more
   or less the same as the original makefile. It can be something like this,
   for building Debug x86:

   ```batch
   SET INCLUDE=$(IncludePath)
   SET LIB=$(LibraryPath)
   MKDIR "$(IntermediateOutputPath)" >NUL 2>&1
   cl.exe /DWIN32 /DNDEBUG /D_CONSOLE /DNO_SSIZE_T /DNO_FPINIT /Fo"$(IntermediateOutputPath)\" src\arithchk.c /nologo /link /OUT:"$(IntermediateOutputPath)arithchk.exe" /NOLOGO
   "$(IntermediateOutputPath)arithchk.exe" >src\arith.h
   del "$(IntermediateOutputPath)arithchk.exe"
   del "$(IntermediateOutputPath)arithchk.obj"
   ```

   For release configurations you must change from _DEBUG to NDEBUG, although
   you can probably just stick with release build of arithcheck.exe even when
   you build your libf2c in debug. In any case, for the configuration building
   libf2c in x64 you must remove the definition of WIN32 from the arguments to
   cl.exe.

   For convenience you can also add the generated header file arith.h to the
   project. You can also add the arithchk.c to the project and set the build
   option to "Exclude From Build", although it does not make a difference for
   the build process.

   Note that NO_SSIZE_T directive is not set by the included VC makefile, but
   it is needed because the typedef of ssize_t, which is a POSIX and not
   standard C type definition, has been removed starting with Visual Studio 2012.

6. Add the following preprocessor directives on project level:

   ```c
   _CRT_SECURE_NO_WARNINGS;USE_CLOCK;MSDOS;NO_ONEXIT;NO_My_ctype;NO_ISATTY
   ```

# For the extra curious

## About ctype.h

The header file ctype.h will be used in the build, even though the source file
ctype.c is not. The file ctype.h has conditional compilation, checking preprocessor
directive NO_My_ctype, for either to include the standard C header with the same
name or for implementing it's macro functions of isdigit and isspace. The source
file ctype.c is only for defining the preprocessor directive My_ctype_DEF which
is checked for in the header to make sure the symbols in the header file are only
defined once (defined when compiling ctype.c, extern-declared when including
ctype.h from other source files). When building with Visual Studio we can use the
standard ctype header, and therefore we define the preprocessor directive
NO_My_ctype. Then compilation of ctype.c is then not needed. Actually ctype.h is
not needed either, so we can delete it from the source directory to make the
compiler search the include paths for the standard ctype header instead. But if
the ctype.h file is present in the source directory it will be used, and then we
must have preprocessor directive NO_My_ctype defined.

## About signed zeros

The description above is for building without the option of
_signed zeros in write statements on IEEE-arithmetic systems_. That requires
some additional files.

# License

The libf2c source code is under copyright by AT&T, Lucent Technologies and Bellcore,
but it's license permits use, copy, modify, and distribute for any purpose and
without fee, provided that the copyright notice is included, and that the
permission notice and warranty disclaimer appear in supporting documentation.
See [LICENSE.md](LICENSE.md).

The Visual Studio project file and other supporting files, including this documention,
is also provided for free use.

# Authors

Albertony