=====================
 Minmake User Manual
=====================

Installation
============
Copy the directory ``scripts`` to the toplevel directory of your
project. ().

Add the following header to the start of your toplevel Makefile::

  # minmake infrastructure, toplevel boilerplate
  PHONY := _all
  _all: all
  MM-sysdir:=$(shell pwd)/scripts
  $(MM-sysdir)/toplevel-init.include: ;
  include $(MM-sysdir)/toplevel-init.include

Add the following footer at the end of your toplevel Makefile::

  # minmake infrastructure, toplevel footer
  $(MM-sysdir)/toplevel-exit.include: ;
  include $(MM-sysdir)/toplevel-exit.include

You can use another name instead of ``scripts``, in that case change
the assignment to ``MM-sysdir`` accordingly.
In order for the auto-invocation feature to work, the parent dir must
be your project's root.

Usage
=====
You can write your makefile as you normally would, minmake just adds
some convenient shortcuts to automatically recurse into
subdirectories and it provides templates and rules using magic
variables that allow a very compact and declarative make style.

Minmake usually runs in summary mode, where make will just issue a
short summary line when it invokes an external command or enters a
subdirectory. You can get the full verbose output by adding ``V=1`` on
the commandline.  Running make -s will work as well and will even
silence the summary.

To only run make in a subtree, set the variable ``D`` on the command
line to the subdirectory, e.g. ``make D=lib/foo`` to only run make in
the lib/foo subtree. This will also work for ``make clean`` and ``make
install``. If ``D`` is used, you may also give some specific target
files, instead of making everything (provided, the target is actually
produced in the given directory, of course).

There is an auto-invocation feature to allow the normal workflow where
you change to a directory and just run ``make`` or ``make foo``.
In order for that feature to work, you will need to
create a symlink named ``GNUmakefile`` in each directory where you
want to use the feature. The symlink must point to minmake's
``call-subdir-make`` file.


Minmake variables and rules
===========================
Minmake works mostly by having some magic variable names that you can
assign values to (or add with ``+=``). Minmake will then automatically
create rules from these using its templates, or do some other action
based on the value. All variables are explained in this section.

In general variables are lists of words separated by whitespace. If
you need to use a filename or directory name that contains spaces
(just don't do it) you can use the special variable ``space``.


Variable 'subdirs'
------------------
The variable ``subdirs`` contains a list of subdirectories where make
is to be called recursively. Each of these subdirectories is
automatically made a phony target. Recursion will also work for the
special target names ``build``, ``clean`` and ``install``.

If the order subdirectories are traversed is important, e.g. if
something is build in subdir foo and later used in subdir bar, simply
declare foo to be a prerequisite for bar::

  bar: foo



Variable 'C-Programs'
---------------------
The words in the variable C-Programs will be the C-Programs to be
built in the current directory.

For each program ``foo`` to be built there will we three other magic
variables:

- ``foo-Objects`` contains the object files foo depends on.
- ``foo-Instdir`` holds the name of the directory the program will be
  installed to by ``make install``. Default is
  $(MM-default-bindir). You may explicitly set that variable to an
  empty value, which will result in the program being built but not
  installed during ``make install``
- ``foo-Staticlibs`` holds the project's static libraries linked into
  foo. In contrast to just using `-lbazlib` in LDLIBS, this will
  relink the binary when the lib changes.

The program will be named ``foo`` and it will built by lining together
all its dependencies. The production rule will also use the variable
INCLUDES that must contain all -I directives for the compiler, and
the variable LOADLIBES that must contain all -L flags to search for libraries.
These two variable are handled specially when using the OUTPUTDIR
feature.

The install target will install to $(DESTDIR)$(PREFIX)$(foo-Instdir)
PREFIX is set to /usr/local by default. DESTDIR is not set and is
indended to be used by package maintainers to produce ``.deb`` or
``.rpm`` packages instead of installing to the local system.

As a convenience shortcurt you may use the value $(all-o-from-c) for
the -Objects variable, which is the list of .c files in the current
directory, with the suffix .c replaced by .o.

Minmake defines its own rules how to produce the needed objet files
from their respective sources, and you should not copy them if you
want to change the rule for some files. The canonic way to change the
production of a certain .o file is to set one of the variables
(CFLAGS, GENERIC_CFLAGS, INCLUDES, DEFINES) in the rule.

Note that minmake will default to use the gcc flags -MD -MP for
compilation of C-Files. This will generate header dependency files
(ending in ``.d``) as a side effect of compilation. These files are
automatically included as well, so a normal make run will recompile
``.c`` files even when only their headers have changed.

If the change is specific to a certain file, use _target specific
variables_ (See section 'How to Use Variables / Target-specific' in the
GNU make manual if you don't know these.)

Here is a complete example for a Makefile using the features mentioned
above. The makefile creates three programs: foo,bar and testfoo, and
generally links against libglib 2.0. ``foo`` is installed to
/usr/local/sbin, ``bar`` to /usr/local/bin`` and testfoo is not
installed at all. Only baz.c is compiled using -DDEBUG. Only testfoo is
linked to libfootest::

  C-Programs := foo bar testfoo

  LDLIBS := -lglib-2.0
  INCLUDES := -I/usr/include/glib-2.0

  foo-Objects := foomain.o baz.o
  foo-Staticlibs := ../lib/libfizz.a
  foo-Instdir := sbin

  bar-Objects := barmain.o baz.o

  testfoo-Objects := testfoo.o
  testfoo-Instdir := 
  
  baz.o: DEFINES += -DDEBUG
  testfoo: LDLIBS += -lfootest


Variable 'CXX-Programs'
-----------------------
The variable is for C++ programs and works in the same way as
``C-Programs`` above. C++ source files are supposed to end in ``.cpp``

There is a convenience shortcut ``$(all-o-from-cpp)`` as well, that
will output a list of object files for all ``.cpp`` files in the
current directory.


Variable 'Static-Libs'
----------------------
Build static libraries. A name ``foo`` in that variable will create
the file ``libfoo.a``, using the object files in ``foo-Objects``.

The library will be installed to
``$(DESTDIR)$(PREFIX)/lib/$(ARCHLIBDIR)`` where $(ARCHLIBDIR) is set
to ``$(shell dpkg-architecture -qDEB_TARGET_MULTIARCH)`` by default.

If the variable ``foo-Headers`` is non empty, it must be a list of
header files. These will be installed to the value of
``foo-Inst-Headers`` or if that is not set, to the default
``$(DESTDIR)$(PREFIX)/include``.


Variable 'Dynamic-Libs'
-----------------------
Build dynamic link libraries. A name ``foo`` will create the file
``libfoo.so.0.0.0`` and will install the lib and the two symlinks
``libfoo.so.0`` ``libfoo.so`` pointing to it. The version can be set
by assigning to ``foo-Version`` The short version can be set by
assigning to ``foo-Sversion``, default is to use the everything until
the first dot from ``foo-Version``.


Variable 'C-Plugins'
--------------------
Plugins are shared objects that are similar to dynamically linked
libraries, but explicitly loaded via dlopen(3). Each name ``foo`` mentioned in
C-Plugins will create a make target ``foo.so`` and you specify its 


Other special variables
-----------------------
There are a few other variables that are used internally, but that you
may use as well:

- ``targets`` Contains all targets made in the current directory when
  make is called without a specific target, i.e. when doing ``make all``.
  These must be real files and they will be removed by ``make clean``.
  Use it when you need to write special recipes not covered by
  minmake.
- ``cleanups`` All files that will be removed during ``make clean``.
  Add files that are produced by your own rules but are not direct
  make targets on their own, e.g. C-header files autogenrated from
  some data source.
- ``PHONY`` Contains all phony targets. These are targets whose
  recipe will be run even if they already exist or appear to be up to
  date. See the `make` manual, section `Phony Targets` for details,
- ``space``, ``comma``, ``empty``, ``squote`` contain what thei name
  suggests and are used to smuggle these characters around make's
  parsing rules.
- Instd

Install other files
-------------------
Installing other files, e.g. static data, default configuration files,
etc. is common enough to warrant a special rule. In the `install`
target you can call the make function ``rule_inst``, with two
arguments, first the filename and second the directory the file is
installed to. (Installing a file under a different name is not
supported.) Example::

  install:
  	@$(call rule_inst,prog.conf,$(DESTDIR)/etc/prog/prog.conf)

The nice side effect calling rule_inst instead of writing your own
install rule is, that it make will issue just a short summary rule as
well, when running ``make install``.

Invocation special vars
-----------------------
A few special variables are intended to be set on the commandline:

- ``V`` sets verbose mode. All commands executed by make are echoed on
  the terminal prior to execution. The default is to just issue a
  short summary for each external command.
- ``D`` sets the subdirectory to descend to. Can be used if you want
  to run make in a subdirectory and not remake the whole project. You
  can not just say ``make -C sub/dir`` because that would not read your
  projects global settings in the toplevel makefile. (Uh, actually you
  _can_ do that if you use some special symlinks, see below).
- ``O`` set the output directory. This will put generated files not in
  the directory containing the source but into a parallel tree rooted
  at the given directory. This mode isn't very well supported and may
  contain bugs. Patches welcome.
