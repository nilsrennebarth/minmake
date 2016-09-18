= minmake =
A minimal make system to use with GNU make and gcc

== Features ==
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

== Usage ==
The files in the `script` directory contain the actual generic code
(and yes, it isn't much) 

Your toplevel Makefile must include the following start and end boilerplate:
~~~~
# make infrastructure, toplevel boilerplate
PHONY := _all
_all: all
SCRIPTDIR:=path to the scripts/ directory
$(SCRIPTDIR)/toplevel-init.include: ;
include $(SCRIPTDIR)/toplevel-init.includ
~~~~

~~~~
# make infrastructure, toplevel footer
$(SCRIPTDIR)/toplevel-exit.include: ;
include $(SCRIPTDIR)/toplevel-exit.include
~~~~

Between the boilerplate, you need to place the project specific defintions.
There are some special magic variables, in particular the variable `subdirs`
which specifies the subdirectories make should descend to.

Otherwise, you mostly set the normal variables that GNU make uses for its
default C compilation rules, e.g. LOADLIBES or some variables defined by
our rules, e.g. GENERIC_CFLAGS, DEFINES, INCLUDES


== How it works ==
The basic trick is, to run make in a subdirectory by calling a special
rule which takes the fixed file descend.include as the real Makefile,
which in turn includes first the generic definitions and second the 
actual path specific makefile, which we could call anyway we like, but
wich is of course named `Makefile`

That magic is coded in descend.include. In generic.include the general
templates for compiling C or C++ programs etc. are defined.

