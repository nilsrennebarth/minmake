=======================
 Minmake Hacking Howto
=======================

Introduction
============
This document should get you started if you want to extend minmakes
capabilities or when you want to find bugs (which hopefully won't
exist :-)

The files in the `scripts` directory contain the actual generic code.

Recurse into subdirectories.
============================

The basic trick is, that directories are transformed into phony
prerequisites.  The rule to make such a prerequisite is then, to run a
submake there which uses ``$(MM-sysdir)/descend.include`` as the makefile

``descend.include`` in turn then includes first the generic
definitions. This will only contain variable and template definitions,
no targets.

Next it includes the actual Makefile in the current
directory (we could name it anyhthing we like, but for convenience we
just use the normal name ``Makefile``).

The it will expand the templates using $(foreach) statements that
iterate over the magic Variables, and finall it adds the production
rules for directory recursion.

Output generation
=================
to be done

Templates
=========
to be done

Various smaller hacks
=====================
to be done
