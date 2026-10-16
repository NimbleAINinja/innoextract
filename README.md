
# innoextract - A tool to unpack installers created by Inno Setup

[Inno Setup](https://jrsoftware.org/isinfo.php) is a tool to create installers for Microsoft Windows applications. innoextract allows to extract such installers under non-Windows systems without running the actual installer using wine. innoextract currently supports installers created by Inno Setup 1.2.10 to 6.3.3.

## About this fork

This is a fork of [dscharrer/innoextract](https://github.com/dscharrer/innoextract) that adds:

* **Inno Setup 6.4.2 support** (always on): recognises the `Inno Setup Setup Data (6.4.2)` ID and reads the `CloseApplicationsFilterExcludes` header string added in 6.4.2.
* **Modified 6.4.2 builds** (opt-in via `INNOEXTRACT_MOD642=1`): some repacks are built with a modified Inno Setup 6.4.2 compiler whose output stock innoextract cannot read. These builds:
  * scramble the CRC32 of every compressed block and block header (deliberately breaking extractors)
  * insert 48 extra (zero) bytes into the setup header before `PasswordTest`
  * store a 16-byte per-file hash instead of SHA-256
  * record the compression method as "stored" while the data chunks actually use LZMA2
  * in some builds, widen `OffsetEXE`, `Offset0` and `Offset1` in the setup loader offset table to 64 bits while keeping table revision 1 and a 32-bit `TotalSize` (neither the stock 6.4 layout, which is all 32-bit, nor the 6.5 one, which is revision 2 with a 64-bit `TotalSize`). innoextract first reads the stock layout and checks that `Offset0` lands on the `Inno Setup Setup Data (` signature. Only if it doesn't does it try this layout, applying the same check, and log `Using modified 6.4.2 loader offset table layout`. A simple bounds check isn't enough: in installers over about 1 GB, the misread stock offsets can still fall inside the file. innoextract stores these offsets as 32-bit values, so installers of 4 GB or more aren't supported.

Without `INNOEXTRACT_MOD642`, behaviour is unchanged from upstream: CRC mismatches are still fatal errors.

### Build (macOS)

    brew install cmake boost xz
    git clone https://github.com/NimbleAINinja/innoextract.git
    cd innoextract && mkdir build && cd build
    cmake .. -DCMAKE_BUILD_TYPE=Release
    make -j8

The binary is `build/innoextract`. Note that `brew install --HEAD innoextract` builds *upstream*, not this fork. The binary links against Homebrew's boost and xz dylibs, so rebuild it if a `brew upgrade` breaks it.

### Run

    # Inspect / list a modified 6.4.2 installer
    INNOEXTRACT_MOD642=1 ./innoextract -i setup.exe
    INNOEXTRACT_MOD642=1 ./innoextract -l setup.exe

    # Extract into a new, empty directory
    mkdir outdir
    INNOEXTRACT_MOD642=1 ./innoextract -e -d outdir setup.exe

Stock 6.4.2 installers need no environment variable.

### What to expect in `INNOEXTRACT_MOD642` mode

* Warnings for `Setup loader checksum mismatch` and `Ignoring block (header) CRC32 mismatch` are expected, as is `Using modified 6.4.2 loader offset table layout` for builds that use the wider offset table.
* If innoextract reports `Could not determine setup data version!` on a 6.4.2 installer, check that `INNOEXTRACT_MOD642=1` is set. Without it, the wider offset table isn't tried.
* You will get one `Checksum mismatch` warning **per file**. The stored per-file hashes don't match MD5, SHA-1 or SHA-256 of the data, so files can't be verified cryptographically. Check the output structurally instead: every file should be its recorded size (innoextract warns `Unexpected output file size` otherwise), and media files should decode (e.g. `afinfo file.wav` on macOS).
* Extraction never runs installer code, but the installer's `[Run]`/`[Code]` sections are not reproduced either. You only get the files.

This mode was verified against six Inno Setup 6.4.2 repacks:

| Installer | Loader table | Files | Size | Result |
|---|---|---|---|---|
| ~226 MB | stock 32-bit layout | 673 | ~344 MB | all files at recorded size; 212/212 WAVs decode |
| ~891 MB | wider 64-bit offsets | 2,077 | ~1.2 GB | all files at recorded size; 440/440 WAVs decode |
| ~894 MB | wider 64-bit offsets | 893 | ~1 GB | all files at recorded size; 282/282 WAVs decode |
| ~921 MB | wider 64-bit offsets | 1,282 | ~1 GB | all files at recorded size; 420/420 WAV/AIFF files decode |
| ~1.8 GB | wider 64-bit offsets | 4,543 | ~2.1 GB | all files at recorded size; 457/457 WAV/AIFF files decode |
| ~2.1 GB | stock 32-bit layout | 2,497 | ~4 GB | all files at recorded size; 650/650 WAVs decode |

After the offset-table fallback was added, the first installer was re-extracted and the output compared byte-for-byte identical to the earlier extraction. The ~1.8 GB installer is the one that needed the signature check: with only a bounds check, the misread stock offsets looked valid and innoextract reported `Could not determine setup data version!`.

## About innoextract

In addition to standard Inno Setup installers, innoextract also supports some modified Inno Setup variants including Martijn Laan's My Inno Setup Extensions 1.3.10 to 3.0.6.1 as well as GOG.com's Inno Setup-based game installers. innoextract is able to unpack Wadjet Eye Games installers (to play with AGS), Arx Fatalis patches (for use with Arx Libertatis) as well as various other Inno Setup executables.

innoextract is available under the ZLIB license - see the LICENSE file.

See the website for [Linux packages](https://constexpr.org/innoextract/#packages).

## Contact

[Website](https://constexpr.org/innoextract/)

Author: [Daniel Scharrer](https://constexpr.org/)

## Dependencies

* **[Boost](https://www.boost.org/) 1.37** or newer
* **liblzma** from [xz-utils](https://tukaani.org/xz/) *(optional)*
* **iconv** (*optional*, either as part of the system libc, as is the case with [glibc](https://www.gnu.org/software/libc/) and [uClibc](https://uclibc.org/), or as a separate [libiconv](https://www.gnu.org/software/libiconv/))

For Boost you will need the headers as well as the `iostreams`, `filesystem`, `date_time`, `system` and `program_options` libraries. Older Boost version may work but are not actively supported. The boost `iostreams` library needs to be build with zlib and bzip2 support.

While innoextract can be built without liblzma by manually setting `-DUSE_LZMA=OFF`, it is highly recommended and you won't be able to extract most installers created by newer Inno Setup versions without it.

To build innoextract you will also need **[CMake](https://cmake.org/) 2.8** and a working C++ compiler, as well as the development headers for liblzma and boost.

See the Website for [operating system-specific instructions](https://constexpr.org/innoextract/install).

## Compile and install

To compile innoextract, run:

    $ mkdir -p build && cd build
    $ cmake ..
    $ make

To install the binaries system-wide, run as root:

    # make install

The default build settings are tuned for users - if you plan to make changes to Arx Libertatis you should append the `-DDEVELOPER=1` option to the `cmake` command to enable debug output and fast incremental builds.

### Build options:

| Option                    | Default   | Description |
|:------------------------- |:---------:|:----------- |
| `BUILD_DECRYPTION`        | `ON`      | Build decryption support.
| `USE_LZMA`                | `ON`      | Use `liblzma`.
| `WITH_CONV`               | *not set* | The charset conversion library to use. Valid values are `iconv`, `win32` and `builtin`¹. If not set, a library appropriate for the target platform will be chosen.
| `CMAKE_BUILD_TYPE`        | `Release` | Set to `Debug` to enable debug output.
| `DEBUG`                   | `OFF`²    | Enable debug output and runtime checks.
| `DEBUG_EXTRA`             | `OFF`     | Expensive debug options.
| `SET_WARNING_FLAGS`       | `ON`      | Adjust compiler warning flags. This should not affect the produced binaries but is useful to catch potential problems.
| `SET_NOISY_WARNING_FLAGS` | `OFF`     | Enable warnings with false positives many cases that still need to be fixed.
| `SET_OPTIMIZATION_FLAGS`  | `ON`      | Adjust compiler optimization flags.
| `CXX_STD_VERSION`         | `2017`    | Maximum C++ standard version to enable.
| `USE_DYNAMIC_UTIMENSAT`   | `OFF`     | Dynamically load utimensat(2) if not available at compile time.
| `USE_STATIC_LIBS`         | `OFF`³    | Turns on static linking for all libraries, including `-static-libgcc` and `-static-libstdc++`. You can also use the individual options below:
| `LZMA_USE_STATIC_LIBS`    | `OFF`⁴    | Statically link `liblzma`.
| `Boost_USE_STATIC_LIBS`   | `OFF`⁴    | Statically link Boost. See also `FindBoost.cmake`.
| `ZLIB_USE_STATIC_LIBS`    | `OFF`⁴    | Statically link `libz`. (used via Boost)
| `BZip2_USE_STATIC_LIBS`   | `OFF`⁴    | Statically link `libbz2`. (used via Boost)
| `iconv_USE_STATIC_LIBS`   | `OFF`⁴    | Statically link `libiconv`.
| `STRICT_USE`              | `OFF`     | Abort if there are missing optional dependencies.
| `DEVELOPER`               | `OFF`     | Enable build options suitable for developers⁵.
| `FASTLINK`                | `OFF`⁶    | Optimize for link speed.
| `USE_LTO`                 | `ON`²     | Use link-time code generation.
| `USE_LD`                  | `best`⁸   | Linker to use - `default`, `mold`, `lld`, `gold`, `bfd` or `best`
| `BUILD_TESTS`             | `OFF`⁶    | Build unit tests that can be run using `make check`
| `RUN_TESTS`               | `OFF`⁷    | Automatically run tests
| `RUN_TARGET`              | (none)    | Wrapper to run binaries produced in the build process
1. The builtin charset conversion only supports Windows-1252 and UTF-16LE. This is normally enough for filenames, but custom message strings (which can be included in filenames) may use arbitrary encodings.
2. Enabled automatically if `CMAKE_BUILD_TYPE` is set to `Debug`.
3. Under Windows, the default is `ON`.
4. Default is `ON` if `USE_STATIC_LIBS` is enabled.
5. Currently this and enables `DEBUG`, `BUILD_TESTS`, `RUN_TESTS` and `FASTLINK` for faster incremental builds and improved debug output, unless those options have been explicitly specified by the user.
6. Enabled automatically if `DEVELOPER` is enabled.
7. Enabled automatically if `DEVELOPER` is enabled unless cross-compiling without `RUN_TARGET` set
8. Disabled automatically (set to `default`) if both `SET_OPTIMIZATION_FLAGS` and `FASTLINK` are disabled. `best` will select the most suited linker based on availability and other settings such as `USE_LTO`.

Install options:

| Option                      | Default              | Description |
|:--------------------------- |:--------------------:|:----------- |
| `CMAKE_INSTALL_PREFIX`      | `/usr/local`         | Where to install innoextract.
| `CMAKE_INSTALL_BINDIR`      | `bin`                | Location for binaries (relative to prefix).
| `CMAKE_INSTALL_DATAROOTDIR` | `share`              | Location for data files (relative to prefix).
| `CMAKE_INSTALL_MANDIR`      | `${DATAROOTDIR}/man` | Location for man pages (relative to prefix).

Set options by passing `-D<option>=<value>` to cmake.

## Run

To extract a setup file to the current directory run:

    $ innoextract <file>

A list of available options can be retrieved using

    $ innoextract --help

Documentation is also available as a man page:

    $ man 1 innoextract

## Limitations

* There is no support for extracting individual components and limited support for filtering by name.

* Included scripts and checks are not executed.

* The mapping from Inno Setup variables like the application directory to subdirectories is hard-coded.

* Names for data slice/disk files in multi-file installers must follow the standard naming scheme.

A perhaps more complete, but Windows-only, tool to extract Inno Setup files is [innounp](http://innounp.sourceforge.net/).

Extracting Windows installer executables created by programs other than Inno Setup is out of the scope of this project. Some of these can be unpacked by the following programs:

* [cabextract](https://cabextract.org.uk/)

* [unshield](https://github.com/twogood/unshield)

## Disclaimer

This project is in no way associated with Inno Setup or [jrsoftware.org](https://jrsoftware.org/).
