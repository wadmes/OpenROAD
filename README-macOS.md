# OpenROAD macOS Build and Usage Guide

This guide covers a local macOS arm64 build of OpenROAD with both the
command-line interface (CUI) and the Qt GUI enabled. It complements the
primary build guide in [docs/user/Build.md](docs/user/Build.md).

The commands below assume they are run from the root of an OpenROAD checkout.

## Prerequisites

- macOS on Apple Silicon.
- Xcode Command Line Tools:

```sh
xcode-select --install
```

- Homebrew in the default Apple Silicon prefix, `/opt/homebrew`.
- Python 3, available as `python3`.
- Git submodules initialized:

```sh
git submodule update --init --recursive
```

If you are starting from a fresh clone, clone recursively instead:

```sh
git clone --recursive https://github.com/The-OpenROAD-Project/OpenROAD.git
cd OpenROAD
```

## Clean Build Flow

Create and activate a Python virtual environment:

```sh
python3 -m venv .venv
source .venv/bin/activate
```

Install the base and common dependencies:

```sh
./etc/DependencyInstaller.sh -base
./etc/DependencyInstaller.sh -common -local
```

Install CUDD 3.0.0 into a checkout-local prefix. The OpenROAD CMake build will
use this prefix through `CUDD_DIR`.

```sh
brew install autoconf automake libtool

export OPENROAD_ROOT="$PWD"
export CUDD_BUILD_DIR="$(mktemp -d /tmp/openroad-cudd-XXXXXX)"

cd "$CUDD_BUILD_DIR"
git clone --depth=1 -b 3.0.0 https://github.com/The-OpenROAD-Project/cudd.git
cd cudd
autoreconf
./configure --prefix="$OPENROAD_ROOT/.local-deps"
make -j "$(sysctl -n hw.logicalcpu)" install

cd "$OPENROAD_ROOT"
```

Build OpenROAD:

```sh
export PATH="$(brew --prefix bison)/bin:$(brew --prefix flex)/bin:$(brew --prefix tcl-tk@8)/bin:${PATH}"
export CMAKE_PREFIX_PATH="$(brew --prefix or-tools)"

./etc/Build.sh -clean -cmake="-DCUDD_DIR=$PWD/.local-deps"
```

The resulting executable is:

```sh
build/bin/openroad
```

## Verify the Build

Check the binary and basic command help:

```sh
build/bin/openroad -version
build/bin/openroad -help
```

Check that the GUI commands are present and supported:

```sh
build/bin/openroad -no_init -no_splash -exit src/gui/test/supported.tcl
```

The GUI support test should print:

```text
Found
Pass
```

For a live GUI smoke test, create a temporary Tcl script:

```sh
cat > /tmp/openroad_gui_smoke.tcl <<'EOF'
if {[info commands gui::supported] ne "::gui::supported"} {
  puts "GUI smoke failed: gui::supported command is missing"
  exit 1
}

if {![gui::supported]} {
  puts "GUI smoke failed: OpenROAD reports GUI is not supported"
  exit 1
}

set rc [catch {
  gui::show {
    if {![gui::enabled]} {
      error "OpenROAD GUI did not report enabled after startup"
    }
    gui::set_title "OpenROAD macOS GUI Smoke"
    gui::hide
  } false false
} msg]

if {$rc != 0} {
  puts "GUI smoke failed: $msg"
  exit 1
}

puts "GUI smoke passed"
exit 0
EOF

build/bin/openroad -no_init -no_splash /tmp/openroad_gui_smoke.tcl
```

The smoke test should open the Qt GUI briefly, close it, and print
`GUI smoke passed`.

## CUI Usage

Print version and command-line help:

```sh
build/bin/openroad -version
build/bin/openroad -help
```

Start the interactive shell:

```sh
build/bin/openroad
```

Run a Tcl script in batch mode and exit when it completes:

```sh
build/bin/openroad -no_init -no_splash -exit path/to/flow.tcl
```

A minimal batch script usually reads technology and design data, runs commands,
and writes results:

```tcl
read_lef path/to/tech.lef
read_def path/to/design.def
report_design_area
write_db results/design.odb
```

Common command-line flags:

- `-no_init`: skip user and site initialization files.
- `-no_splash`: suppress the startup banner.
- `-threads <N>`: set the worker thread count.
- `-log <file>`: write the OpenROAD log to a file.
- `-metrics <file>`: write metrics to a file.
- `-db <file>`: load an OpenDB database.
- `-exit`: exit after the script finishes.

## GUI Usage

Start the GUI:

```sh
build/bin/openroad -gui
```

Start the GUI and load an OpenDB database:

```sh
build/bin/openroad -gui -db path/to/design.odb
```

Run a Tcl script, then open the GUI from Tcl:

```tcl
read_db path/to/design.odb
gui::show
```

Useful GUI Tcl commands:

- `gui::show`: open the GUI.
- `gui::enabled`: return whether the GUI is currently active.
- `gui::fit`: fit the loaded design in the layout view.
- `gui::hide`: close the GUI and return control to Tcl.
- `gui::selection_add_net <net>`: select a net.
- `gui::selection_add_inst <inst>`: select an instance.
- `gui::highlight_net <net> [highlight_group]`: highlight a net.
- `gui::highlight_inst <inst> [highlight_group]`: highlight an instance.

See [src/gui/README.md](src/gui/README.md) for the complete GUI command list.

## Troubleshooting

### Homebrew Permission Or Cache Errors

Run Homebrew as your normal user, not with `sudo`. If Homebrew reports
permission problems under `/opt/homebrew` or its cache directories, fix the
ownership of the affected paths and rerun the dependency command.

```sh
brew doctor
brew update
```

### Qt5 CMake Path Issues

Install Qt 5 with Homebrew:

```sh
brew install qt@5
```

The macOS build should pass Qt to CMake as a distinct option:

```sh
-DQt5_DIR=$(brew --prefix qt@5)/lib/cmake/Qt5
```

If CMake reports that it cannot find Qt5, confirm that the local
`etc/Build.sh` macOS block appends CMake options as separate array entries.
String-appended CMake options can be split incorrectly by the shell.

### Miniconda Boost Shadowing Homebrew Boost

If Miniconda or another Python distribution is ahead of Homebrew in `PATH`,
CMake may pick up the wrong Boost package. Prefer Homebrew Boost for this
build:

```sh
brew install boost
export Boost_ROOT="$(brew --prefix boost)"
export Boost_DIR="$(find "$(brew --prefix boost)/lib/cmake" -maxdepth 1 -type d -name 'Boost-*' | head -n 1)"
```

If needed, temporarily move Conda paths after Homebrew paths before running the
build.

### Missing autoreconf

CUDD needs `autoreconf` to generate its configure script. Install the
Homebrew autotools packages:

```sh
brew install autoconf automake libtool
```

### Missing CUDD

If CMake reports that it cannot find CUDD, confirm that the checkout-local
install exists:

```sh
test -f .local-deps/include/cudd.h
test -f .local-deps/lib/libcudd.a
```

Then rebuild with:

```sh
./etc/Build.sh -clean -cmake="-DCUDD_DIR=$PWD/.local-deps"
```

## Local Machine Appendix

The local macOS GUI verification for this checkout used:

- OpenROAD version: `26Q2-1257-ga86247f4e8`.
- Python virtual environment: `.venv`.
- CUDD install prefix: `.local-deps`.
- Build command:

```sh
source .venv/bin/activate
export PATH="$(brew --prefix bison)/bin:$(brew --prefix flex)/bin:$(brew --prefix tcl-tk@8)/bin:${PATH}"
export CMAKE_PREFIX_PATH="$(brew --prefix or-tools)"
./etc/Build.sh -clean -cmake="-DCUDD_DIR=$PWD/.local-deps"
```

The local macOS build context also preserved these `etc/Build.sh` fixes:

- Pass macOS CMake options as separate array entries.
- Use `TCL_HEADER` for Tcl header discovery.
- Prefer Homebrew Boost over a Miniconda-provided Boost package.

The following checks passed locally:

```sh
build/bin/openroad -version
build/bin/openroad -help
build/bin/openroad -no_init -no_splash -exit src/gui/test/supported.tcl
build/bin/openroad -no_init -no_splash /tmp/openroad_gui_smoke.tcl
```
