Directory Structure
*******************

The MicroPython repository is divided into a number of directories, each with
its own function. Some of them include:


Docs, Logo and Examples
=======================

The user documentation for all ports lives in the ``docs`` directory, with some
example code in the ``examples`` directory. The ``logo`` directory contains the
MicroPython logo in various formats.


Tests
=====

All tests live in the ``tests`` directory.


Python Code
===========

The ``py`` directory contains the bulk of code for the compiler, runtime
environment and core library of MicroPython. This is where we will be looking
for API functions and macros.


Port-specific Code
==================

The directories such as ``esp8266``, ``cc3200``, ``pic16bit``, ``teensy``,
``stmhal``, ``bare-arm``, ``qemu-arm``, ``unix`` and ``windows`` contain code
and tools specific to particular ports of MicroPython. If you are working on a
feature to be added for a specific hardware, this is probably the best place to
put your files.


Minimal Port
============

The ``minimal`` directory contains the minimum of files that a new port needs.
You can use this as a template for starting new ports of MicroPython.


Common Parts
============

The ``extmod`` directory contains source of MicroPython modules that may are
not part of the core library, but are useful for multiple ports. The
``drivers`` directory has code for libraries that communicate with various
additional hardware, such as displays or sensors.


Tools and Utilities
===================

The ``tools`` directory contains various tools useful for working with
MicroPython. The ``mpy-cross`` directory has code for the MicroPython
cross-compiler, which can be used to generate frozen bytecode modules.
