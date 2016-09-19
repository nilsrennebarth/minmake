# minmake #
A minimal make system to use with GNU make and gcc

## Features ##
The make system uses GNU make's (and GCC's) features to create a build system
with the following features:
- recursive builds
- common options (compile flags, defines, ...) can be set at the
  root of the project
- in general, options can be set in every directory and propagate to
  subdirectories
- various levels of verbosity:
  - quiet
  - just show what files are generated
  - show the full commands as make traditionally does
- support a pristine source build where all generated files are created
  outside of the source tree.
- Completely automated generation of all C source dependencies
- Support building of static and dynamic libraries and normal
  programs.
- Small Makefiles with a minimum of boilerplate

## Usage ##
minmake assumes that all actual code is contained in subdirectories of the
project's root directory. The root directory Makefile needs to contain a
certain header at the beginning and a footer at the end to pull in
minmake's definitions. In between you place project wide definitions
in the same way as in a normal Makefile. There are some *magic* variables
where assignments have special meanings, and there are some magic targets as
well. Otherwise, you mostly set the normal variables that GNU make uses for its
default C compilation rules, e.g. LOADLIBES or some variables defined by
our rules, e.g. GENERIC_CFLAGS, DEFINES, INCLUDES

Also you need to add the `scripts` directory (together with its contents of
course) to your project's root directory.

Here are header and footer for the toplevel Makefile:

### Toplevel header ###
~~~~
# minmake infrastructure, toplevel boilerplate
PHONY := _all
_all: all
SCRIPTDIR:=$(CURDIR)/scripts
$(SCRIPTDIR)/toplevel-init.include: ;
include $(SCRIPTDIR)/toplevel-init.include
~~~~

### Toplevel footer ###
~~~~
# minmake infrastructure, toplevel footer
$(SCRIPTDIR)/toplevel-exit.include: ;
include $(SCRIPTDIR)/toplevel-exit.include
~~~~

### Magic variable: subdirs ###
The variable `subdirs` must be a list of all subdirectories where make is to
be called recursively.

Each of these subdirectories is automatically made a phony target.

If make must be run in some subdirectories first (e.g. because these build a
static library used for linkin in others), just declare normal dependencies
between the subdirectories, without any attached rules.

### Magic variable: C-Programs ###
The (white space separated) words in the variable `C-Programs` will be the
C-Programs to be built in the corresponding directory.

For each program *prog* to be built there will we another magic variable named
*prog*`-Objects`, and assignment to that variable specifies the object files
(and static libraries) linked together for that program. Corresponding pseudo
targets `clean` and `install` will be created as well.
As a convenience shortcurt you may use the value `$(all-o-from-c)` which is
the list of `.c` files in that directory, with the suffix `.c` replaced by
`.o`.

The variable `INCLUDES` must contain all -I directives for the compiler,
the variable `LOADLIBES` must contain all -L flags to search for libraries.
To compile a certain source file with different flags use target specific
variables (See the GNU make manual if you don't know these. In short you
specify the target, a colon and then a list of variable assignments.)

The `install` target will install foo to $(DESTDIR)$(PREFIX)/bin and PREFIX is
set to `/usr/local` by default. That default can be overridden in the toplevel
Makefile. The part `bin` is just the default. In case a specific program is
to be installed to another directory use the magic Variable *prog*-Instdir

#### Example ####
The following Makefile in a subdirectory will create the binary foo from 
main.o, bar.o and baz.o, each of those compiled from its respective source as usual.
When compiling baz.c, the additional preprocessor symbol DEBUG will be
defined. The program will also be linked statically against the library
libfizz.a from the sibling directory `lib`.
`foo` will be installed to `/usr/local/sbin/foo` by the install target.

~~~~
C-Programs := foo
foo-Objects := main.o bar.o baz.o ../lib/libfizz.a
foo-Instdir := sbin

baz.o: DEFINES += -DDEBUG
~~~~

### Magic variable CXX-Programs ###
This is like the variable `C-Programs`, but for C++ sources. There is a shortcut
for all c++ source files as well: `$(all-o-from-cpp)`, which assumes that c++
source files end in `.cpp`

### Magic variable Static-Libs ###
The words in the magic variable `Static-Libs` are the static libs to be built in
the corresponding directory. For each library *libl*, the magic variable
*libl*`-Objects` will specify the object files of the library.

### Magic variable Dynamic-Libs ###
`Dynamic-Libs` will produce DLLs, with the object files for library *libl*
specified by the magic variable *libl*`-Objects`. The magic variable
*libl*`-Version` sets the version number, the magic variable *libl*`-Sversion`
sets the major version, which by default is the version up to the first dot.

## How it works ##
The files in the `script` directory contain the actual generic code
(and yes, it isn't much) 


The basic trick is, that directories are added as prerequisites (marked by
ending in `/`) The rule to make such a prerequisite is then, to run a submake
there which uses descend.include as makefile, (which is found because the
`scripts` directory is added to make's search path using `--include-dir`
descend.include in turn then includes first the generic definitions and then
the actual makefile in the subdirectory, which we could name anything we like,
but for simplicity just name it `Makefile`

That magic is coded in descend.include. In generic.include the general
templates for compiling C or C++ programs etc. are defined.

## Thanks ##
Thanks to the designers of the the Linux Kernel build system, which is where
I took the basic ideas from.
