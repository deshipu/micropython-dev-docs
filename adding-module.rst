Adding a Module
***************

When using MicroPython, you have even bigger chance to need your own library
written in C than in other Python implementations. You might want to use one of
the existing libraries for your platform, get access to some peripherals or
other features of the hardware that is not exposed to Python by default, or
even just do something faster and with smaller memory overhead. To do that, you
will need to extend the existing firmware with your own C code, and this
article is going to show you how to do it, using the ESP8266 port as the
example.


Adding Your Own Source File
===========================

In order to add your own MicroPython module written in C, you need to create a
new C file and add references to it to several files, so that the file is
picked up during the compilation.

First of all, you need to add it to the ``Makefile``, to the list of source
files in the ``SRC_C`` variable. Make sure to follow the same format as the
other files in there. The order doesn't matter much.

.. code-block:: make

    SRC_C = \
            main.c \
            system_stm32.c \
            stm32_it.c \
            ...
            mymodule.c
            ...

The second file you will need to add to is ``esp8266.ld``, which is the map of
memory used by the compiler. You have to add it to the list of files to be put
in the ``.irom0.text`` section, so that your code goes into the instruction
read-only memory (iROM). If you fail to do that, the compiler will try to put
it in the instruction random-access memory (iRAM), which is a very scarce
resource, and which can get overflown if you try to put too much there.

Now just create an empty ``mymodule.c`` file, and run the compilation to see
that it is now included in the firmware.


Creating a Python Module
========================

We have our file, but it doesn't actually do anything. It's empty, and there
is no new python module that we could import. Time to change that.

From the C side, modules in MicroPython are simply structs with a certain
structure. Open the ``mymodule.c`` file and put this code inside:


.. code-block:: c

    #include "py/nlr.h"
    #include "py/obj.h"
    #include "py/runtime.h"
    #include "py/binary.h"
    #include "portmodules.h"


    STATIC const mp_map_elem_t mymodule_globals_table[] = {
        { MP_OBJ_NEW_QSTR(MP_QSTR___name__), MP_OBJ_NEW_QSTR(MP_QSTR_mymodule) },
    };

    STATIC MP_DEFINE_CONST_DICT(mp_module_mymodule_globals, mymodule_globals_table);

    const mp_obj_module_t mp_module_mymodule = {
        .base = { &mp_type_module },
        .name = MP_QSTR_mymodule,
        .globals = (mp_obj_dict_t*)&mp_module_mymodule_globals,
    };

What does this code do? It just defines a python module, using
``mp_obj_module_t`` type, and then initializes some of its fields, such as the
base type, the name, and the dictionary of globals for that module. In that
dictionary, it defines one variable, ``__name__``, with the name of our module
in it. That's it.

Now, for this module to actually be available for import, we need to add it to
``mpconfigport.h`` file to ``MICROPY_PORT_BUILTIN_MODULES``:

.. code-block:: c

    extern const struct _mp_obj_module_t mp_module_mymodule;

    #define MICROPY_PORT_BUILTIN_MODULES \
        { MP_OBJ_NEW_QSTR(MP_QSTR_umachine), (mp_obj_t)&machine_module }, \
        ...
        { MP_OBJ_NEW_QSTR(MP_QSTR_mymodule), (mp_obj_t)&mp_module_mymodule }, \

Now you can try compiling the firmware and flashing it to your board. Then you
can run ``import mymodule`` and see it imported.


Adding a Function
=================

Now let's add a simple function to that module. Edit ``mymodule.c`` again and
add this code right after the includes:

.. code-block:: c

    #include <stdio.h>


    STATIC mp_obj_t mymodule_hello(void) {
        printf("Hello world!\n");
        return mp_const_none;
    }
    STATIC MP_DEFINE_CONST_FUN_OBJ_0(mymodule_hello_obj, mymodule_hello);


This creates a function object ``mymodule_hello_obj`` which takes no arguments,
and when called, executes the C function ``mymodule_hello``. Also note, that
our function has to return something -- so we return ``None``. Now we need to
actually add that function object to our module:

.. code-block:: c

    STATIC const mp_map_elem_t mymodule_globals_table[] = {
        { MP_OBJ_NEW_QSTR(MP_QSTR___name__), MP_OBJ_NEW_QSTR(MP_QSTR_mymodule) },
        { MP_OBJ_NEW_QSTR(MP_QSTR_hello), (mp_obj_t)&mymodule_hello_obj },
    };

Now when you compile and flash the firmware, you will be able to import the
module and call the function inside it.


Function Arguments
==================

The ``MP_DEFINE_CONST_FUN_OBJ_0`` macro that we used to define our function is
a shortcut for defining a function with no arguments. We can also define a
function that takes a single argument with ``MP_DEFINE_CONST_FUN_OBJ_1`` -- the
C function then needs to take an argument of type ``mp_obj_t``:

.. code-block:: c

    STATIC mp_obj_t mymodule_hello(mp_obj_t what) {
        printf("Hello %s!\n", mp_obj_str_get_str(what));
        return mp_const_none;
    }
    STATIC MP_DEFINE_CONST_FUN_OBJ_1(mymodule_hello_obj, mymodule_hello);


Note that the ``mp_obj_str_get_str`` function will automatically raise the
right exception on the python side if the argument we gave it is not a python
string.  This is very convenient.

It's also possible to define functions with variable number of arguments, or
even with keyword arguments -- you can easily find examples of that in the
modules already included in MicroPython. I will not be covering this in detail.


Classes
=======

Let's try to add a class to our module. A class is similar to a module -- it's
also a C struct with certain fields:

.. code-block:: c

    STATIC const mp_rom_map_elem_t mymodule_hello_locals_dict_table[] = {
    }
    STATIC MP_DEFINE_CONST_DICT(mymodule_hello_locals_dict,
                                mymodule_hello_locals_dict_table);

    const mp_obj_type_t mymodule_hello_type = {
        { &mp_type_type },
        .name = MP_QSTR_Hello,
        .print = mymodule_hello_print,
        .make_new = mymodule_hello_make_new,
        .locals_dict = (mp_obj_dict_t*)&mymodule_hello_locals_dict,
    };

It needs two functions: one for creating the class and allocating all the
memory it needs, and one for printing the objects of that class (similar to
python's ``__repr__``). Let's add them near the top of our file:

.. code-block:: c

    typedef struct _mymodule_hello_obj_t {
        mp_obj_base_t base;
        uint8_t hello_number;
    } mymodule_hello_obj_t;


    mp_obj_t mymodule_hello_make_new(const mp_obj_type_t \*type, size_t n_args,
                                     size_t n_kw, const mp_obj_t \*args) {
        mp_arg_check_num(n_args, n_kw, 1, 1, true);
        pyb_spi_obj_t \*self = m_new_obj(mymodule_hello_obj_t);
        self->base.type = &mymodule_hello_type;
        self->hello_number = mp_obj_get_int(args[0])
        return MP_OBJ_FROM_PTR(self);
    }


    STATIC void pyb_spi_print(const mp_print_t \*print, mp_obj_t self_in, mp_print_kind_t kind) {
        pyb_spi_obj_t \*self = MP_OBJ_TO_PTR(self_in);
        mp_printf(print, "Hello(%u)", self->hello_number);
    }


We define a struct to hold all our class data, with one additional field
``hello_number``. Then we define functions to create and to print it.

Of course you also need to add the class to the module's globals. Compile it
and try creating and printing objects of our new class.


Adding Methods
==============

Methods in MicroPython are just functions in the class's locals dict. You add
them the same way as you do to modules, just remember that the first argument
is a pointer to the data struct:

.. code-block:: c

    STATIC mp_obj_t mymodule_hello_increment(mp_obj_t self_in) {
        pyb_spi_obj_t \*self = MP_OBJ_TO_PTR(self_in);
        self->hello_number += 1;
        return mp_const_none;
    }
    MP_DEFINE_CONST_FUN_OBJ_1(mymodule_hello_increment_obj,
                              mymodule_hello_increment);


Also, don't forget to add them to the locals dict:

.. code-block:: c

    STATIC const mp_rom_map_elem_t mymodule_hello_locals_dict_table[] = {
        { MP_ROM_QSTR(MP_QSTR_read), MP_ROM_PTR(&mymodule_hello_increment_obj) },
    }

And that's all.
