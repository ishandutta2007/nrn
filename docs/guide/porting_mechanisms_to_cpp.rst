.. include:: ../rst_substitutions.txt

.. _porting-mechanisms-to-cpp:

Adapting MOD files for C++ with |neuron_with_cpp_mechanisms|
==========================================================

In older versions of NEURON, MOD files containing NMODL code were translated
into C code before being compiled and executed by NEURON.
Starting with |neuron_with_cpp_mechanisms|, NMODL code is translated into C++
code instead.

In most cases, this does not present any issues, as simple C code is typically
valid C++, and no changes are required.
However, C and C++ are not the same language, and there are cases in which MOD
files containing ``VERBATIM`` blocks need to be modified in order to build with
|neuron_with_cpp_mechanisms|.

Before you start, you should decided if you need your MOD files to be
compatible simultaneously with |neuron_with_cpp_mechanisms| **and** older, or
if you can safely stop supporting older versions.
Supporting both is generally possible, but it may be more cumbersome than
committing to using C++ features.
Considering NEURON has maintained strong backward compatibility and internal
numerical methods haven't changed with migration to C++, it would be sufficient
to adapt your MOD files to C++ only and use |neuron_with_cpp_mechanisms|.

.. note::
  If you have a model that stopped compiling when you upgraded to or beyond
  |neuron_with_cpp_mechanisms|, the first thing that you should check is
  whether the relevant MOD files have already been updated in ModelDB or in the
  GitHub repository of that model. You can check the repository name with the
  model accession number under `<https://github.com/ModelDBRepository>`_.
  An updated version may already be available!

..
  Does this need some more qualification? Are there non-VERBATIM
  incompatibilities?

Legacy patterns that are invalid C++
------------------------------------

This section aims to list some patterns that the NEURON developers have found
in pre-|neuron_with_cpp_mechanisms| models that need to be modified to be valid
C++.

Implicit pointer type conversions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
C++ has stricter rules governing pointer type conversions than C. For example

.. code-block:: cpp

  double* x = (void*)0;   // valid C, invalid C++
  double* x = nullptr;    // valid C++, invalid C
  double* x = (double*)0; // valid C and C++ (C-style casts discouraged in C++)

Similarly, in C one can pass a ``void*`` argument to a function that expects a
``double*``. In C++ this is forbidden.

The same issue may manifest itself in code such as

.. code-block:: cpp

  double* x = malloc(7 * sizeof(double));          // valid C, invalid C++
  double* x = new double[7];                       // valid C++, invalid C
  double* x = (double*)malloc(7 * sizeof(double)); // valid C and C++ (C-style casts discouraged in C++)

If you choose to move from using C ``malloc`` and ``free`` to using C++ ``new``
and ``delete`` then remember that you cannot mix and match ``new`` with ``free``
and so on.

.. note::
  Explicit memory management with ``new`` and ``delete`` is discouraged in C++
  (`R.11: Avoid calling new and delete explicitly <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rr-newdelete>`_).
  If you do not need to support older versions of NEURON, you may be able to
  use standard library containers such as ``std::vector<T>``.

.. _local-non-prototype-function-declaration:

Local non-prototype function declarations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
In C, the function declaration

.. code-block:: c

  void some_function();

is a `non-prototype function declaration
<https://en.cppreference.com/w/c/language/function_declaration>`_: it declares
that ``some_function`` exists, but it does not specify the number of arguments
that it takes, or their types.
In C++, the same code declares that ``some_function`` takes zero arguments (the
C equivalent of this is ``void some_function(void)``).

If such a declaration occurs in a top-level ``VERBATIM`` block then it is
likely to be harmless: it will add a zero-parameter overload of the function,
which will never be called.
It **will**, however, cause a problem if the declaration is included **inside**
an NMODL construct

.. code-block:: c

  PROCEDURE procedure() {
  VERBATIM
    void some_method_taking_an_int(); // problematic in C++
    some_method_taking_an_int(42);
  ENDVERBATIM
  }

because in this case the local declaration hides the real declaration, which is
something like

.. code-block:: c

  void some_method_taking_an_int(int);

in a NEURON header that is included when the translated MOD file is compiled.
In this case, the problematic local declaration can simply be removed.

.. _function-decls-with-incorrect-types:

Function declarations with incorrect types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
In older MOD files and versions of NEURON, API methods were often accessed by
declaring them in the MOD file and not by including a correct declaration from
NEURON itself.
In |neuron_with_cpp_mechanisms|, more declarations are implicitly included when
MOD files are compiled. This can lead to problems if the declaration in the MOD
file did not specify the correct argument and return types.

.. code-block:: cpp

  Rand* nrn_random_arg(int); // correct declaration from NEURON header, only in new NEURON
  /* ... */
  void* nrn_random_arg(int); // incorrect declaration in MOD file, clashes with previous line

The fix here is simply to remove the incorrect declaration from the MOD file.

If the argument types are incorrect, the situation is slightly more nuanced.
This is because C++ supports overloading functions by argument type, but not by
return type.

.. code-block:: cpp

  void sink(Thing*); // correct declaration from NEURON header, only in new NEURON
  /* ... */
  void sink(void*); // incorrect declaration in MOD file, adds an overload, not a compilation error
  /* ... */
  void* void_ptr;
  sink(void_ptr);  // probably used to work, now causes a linker error
  Thing* thing_ptr;
  sink(thing_ptr); // works with both old and new NEURON

Here the incorrect declaration ``sink(void*)`` declares a second overload of
the ``sink`` function, which is not defined anywhere.
With |neuron_with_cpp_mechanisms| the ``sink(void_ptr)`` line will select the
``void*`` second overload, based on the argument type, and this will fail during
linking because this overload is not defined (only ``sink(Thing*)`` has a
definition).

In contrast, ``sink(thing_ptr)`` will select the correct overload in
|neuron_with_cpp_mechanisms|, and it also works in older NEURON versions
because ``Thing*`` can be implicitly converted to ``void*``.

The fix here is, again, to remove the incorrect declaration from the MOD file.

See also the section below, :ref:`deprecated-overloads-taking-void`, for cases
where NEURON **does** provide a (deprecated) definition of the ``void*``
overload.

K&R function definitions
^^^^^^^^^^^^^^^^^^^^^^^^
C supports a legacy ("K&R") syntax for function declarations. This is slated
for removal from the C language, and is not valid C++.

There is no advantage to the legacy syntax. If you have legacy definitions such
as

.. code-block:: c

  void foo(a, b) int a, b; { /* ... */ }

then simply replace them with

.. code-block:: cpp

  void foo(int a, int b) { /* ... */ }

which is valid in both C and C++.

Legacy patterns that are considered deprecated
----------------------------------------------
As noted above (:ref:`local-non-prototype-function-declaration`), declarations
such as

.. code-block:: c

  VERBATIM
  extern void vector_resize();
  ENDVERBATIM

at the global scope in MOD files declare C++ function overloads that take no
parameters.
If such declarations appear at the global scope then they do not hide the
correct declarations, so this can be harmless, but it is not necessary.
In |neuron_with_cpp_mechanisms| the full declarations of the NEURON API methods
that can be used in MOD files are implicitly included via the ``mech_api.h``
header, so this explicit declaration of a zero-parameter overload is not needed
and can safely be removed.

.. _deprecated-overloads-taking-void:

Deprecated overloads taking ``void*``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
As noted above (:ref:`function-decls-with-incorrect-types`),
|neuron_with_cpp_mechanisms| provides extra overloads for some API methods that
aim to smooth the transition from C to C++.
These overloads are provided for widely used methods, and where overloading on
return type would not be required.

An example is the ``vector_capacity`` function, which in
|neuron_with_cpp_mechanisms| has two overloads

.. code-block:: cpp

  int vector_capacity(IvocVect*);
  [[deprecated("non-void* overloads are preferred")]] int vector_capacity(void*);

The second one simply casts its argument to ``IvocVect*`` and calls the first
one.
The ``[[deprecated]]`` attribute means that MOD files that use the second
overload emit compilation warnings when they are compiled using ``nrnivmodl``.

If your MOD files produce these deprecation warnings, make sure that the
relevant method (``vector_capacity`` in this example) is being called with an
argument of the correct type (``IvocVect*``), and not a type that is implicitly
converted to ``void*``.
