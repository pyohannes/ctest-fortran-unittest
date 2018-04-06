Unit testing with Fortran and CTest
===================================

Fortran, one of the oldest programming languages, is alive. This was confirmed
to me recently, when I had to devise a unit test framework for a Fortran
project. While I am an avid supporter of comprehensive unit testing I advocate 
a minimalistic approach towards unit testing frameworks, which is reflected in
the approach described below.

This post is a step-by-step through setting up a working sample CMake project.
While all commands in this post are meant to be executed on a Unix shell the
CMake project itself is also supposed to work with Visual Studio on Windows.

Why CTest?
----------

We decided to use CTest for the following reasons:

- *No dependencies.* CMake was already used to build the Fortran project. So wherever the project
  can be built, CTest is available too. No additional dependencies are
  necessary.
- *Easy to integrate.* CTest was already used for C/C++ unit tests and could be well integrated
  into the toolchain (test results are reported via Jenkins).
- *Simplicity.* The concept of plain CTest tests is easy enough to grasp for Fortran
  programmers who do not have a CS degree. There is no need to learn
  the peculiarities of a fancy unit testing framework.

Though I like the minimalism of plain CTest and the freedom that comes with it,
I am well aware of the danger that comes with the lack of rigidity and control.
However, experience shows that if one makes it hard for developers to write
tests, no tests get written. 

How to write tests
------------------

Using plain CTest and relying on the CMake function ``create_test_sourcelist``,
a test has to meet the following requirements:

#. Every test function must be placed in its own file.
#. The name of the test function must conform to the name of the test file
   relative to the CMake project. The file extension is omitted and slashes are
   replaced by underscores.
#. On success, the test function has to return 0, on failure it has to return
   a value different from 0.

The first two requirements can automatically be ensured by offering a utility 
that creates files with empty test templates. Meeting the third requirement 
can be supported by providing simple assertion macros.

For further illustration I will use two test cases, one failing and one
succeeding. Both reside in a subdirectory ``tests``. The successful test returns
0:

.. code::

    $ cat > tests/test_succeeds_1.f90 <<EOF
    integer function tests_test_succeeds_1() result(r)
        r = 0
    end function
    EOF
    $

The failing test returns 1:

.. code::

    $ cat > tests/test_failing_1.f90 <<EOF
    integer function tests_test_failing_1() result(r)
        r = 1
    end function
    EOF
    $

The correspondence between file name and test function name is crucial here. By
discarding the file extension and replacing slashes with underscores, the file
named ``tests/test_succeeds_1.f90`` is supposed to contain a test function called
``tests_test_succeeds_1``.

CMakeLists.txt
--------------

I will now go step-by-step throught the ``CMakeList.txt`` file that sets up the
Fortran test suite. I starts out with the boilerplate code specifying the CMake
version and enabling needed languages and CTest support. C is required as C
code will be created for the test driver executable:

.. code-block:: cmake

    cmake_minimum_required (VERSION 3.5)
    enable_language (C Fortran)
    enable_testing ()

In a next step Fortran name mangling has to be considered. This is necessary,
as the call ``create_test_sourcelist`` deduces symbol names from file names.
Regarding Fortran, those names are different between platforms and compilers:
Intel Fortran on Windows produces uppercase symbol names, Intel Fortran and GNU
Fortran on Linux produce lowercase symbol names with a trailing underscore:

+----------------------+-------------------------------+----------------------------+
| File name            | ``tests/test_succeeds_1.f90`` | ``tests/test_FAILS_1.f90`` | 
+----------------------+-------------------------------+----------------------------+
| Function name        | ``tests_test_succeeds_1``     | ``tests_test_FAILS_1``     | 
+----------------------+-------------------------------+----------------------------+
| Intel Windows symbol | ``TESTS_TEST_SUCCEEDS_1``     | ``TESTS_TEST_FAILS_1``     | 
+----------------------+-------------------------------+----------------------------+
| GNU Linux symbol     | ``tests_test_succeeds_1_``    | ``tests_test_fails_1_``    |
+----------------------+-------------------------------+----------------------------+

The function ``mangle_fortran_name`` returns the mangled symbol name for a
Fortran function name. The code below works for Intel Fortran on Windows and
for Intel and GNU Fortran on Linux. It might be necessary to extend it for
other compilers:

.. code-block:: cmake

    function (mangle_fortran_name CNAME FNAME)
        set (TMP)
        if (WIN32)
            string (TOUPPER "${FNAME}" TMP)
        else ()
            string (TOLOWER "${FNAME}_" TMP)
        endif ()
        set (${CNAME} ${TMP} PARENT_SCOPE)
    endfunction ()
   
The function ``mangle_fortran_filename_list`` takes a list of valid file names
and returns a list of mangled names that then can be used to fool
``create_test_sourcelist`` lateron. The extension of every file name is
discarded and the remaining part is mangled. This does not result in the real
symbol name, as the result can still contain slashes. The slashes are not
removed as this point, as ``create_test_sourcelist`` can take care of this by
itself.
    
.. code-block:: cmake

    function (mangle_fortran_filename_list MANGLED)
        set (TMP)
        foreach (TFILE ${ARGN})
            string (REGEX REPLACE ".f90$" "" TESTNAME ${TFILE})
            mangle_fortran_name (C_TESTNAME ${TESTNAME})
            list (APPEND TMP ${C_TESTNAME})
        endforeach ()
        set (${MANGLED} ${TMP} PARENT_SCOPE)
    endfunction()
   
Now comes the final function that creates the test driver target and registers
tests. It accepts two arguments (a target name and a list of test file names)
and does the following:

#. Creates a list ``TEST_FILES`` containing valid file names of tests.
#. Creates a list ``TEST_FILES_MANGLED`` containing mangled names of tests
   files.
#. A call to ``create_test_sourcelist`` with the mangled test file names
   creates test driver code in ``main.c``.
#. Fortran objects are compiled into a static library, which is then linked
   into the test driver executable. The separate Fortran library is necessary
   for a seamless Visual Studio integration.
#. Finally tests are registered for CTest by calling ``add_test`` in a loop
   that iterates over ``TEST_FILES`` and ``TEST_FILES_MANGLED`` simultaneously. 
   The valid file names are used as test names, as they are more informative 
   than mangled names.  

.. code-block:: cmake

    function (add_fortran_test_executable TARGET)
        set (TEST_FILES ${ARGN})
        mangle_fortran_filename_list (TEST_FILES_MANGLED ${TEST_FILES})
    
        create_test_sourcelist (_ main.c ${TEST_FILES_MANGLED})
    
        add_library (${TARGET}_fortran ${TEST_FILES})
        add_executable (${TARGET} main.c)
        target_link_libraries (${TARGET} ${TARGET}_fortran)
    
        set (INDEX 0)
        list (LENGTH TEST_FILES LEN)
        while (${LEN} GREATER ${INDEX})
            list (GET TEST_FILES ${INDEX} TEST_FILE)
            list (GET TEST_FILES_MANGLED ${INDEX} TEST_FILE_MANGLED)
            add_test (
                NAME ${TEST_FILE}
                COMMAND $<TARGET_FILE:${TARGET}> ${TEST_FILE_MANGLED})
            math (EXPR INDEX "${INDEX} + 1")
        endwhile ()
    endfunction ()

In the end, all the dirty hacking is hidden behind the call to 
``add_fortran_test_executable``, where one only needs to specify the target 
name and the list of test files:   
    
.. code-block:: cmake

    add_fortran_test_executable (
        testsuite
        "tests/test_fails_1.f90"
        "tests/test_succeeds_1.f90")

Dependencies can now be linked to the resulting ``testsuite`` and, if
necessary, to the ``testsuite_fortran`` target. The latter might be necessary
to account for Fortran module dependencies.


Running the tests
-----------------

The project can now be built and tests are run via ctest:

.. code-block::

    $ find .
    ./CMakeLists.txt
    ./tests/test_fails_1.f90
    ./tests/test_succeeds_1.f90
    $ mkdir bld
    $ cd bld
    $ cmake .. > /dev/null
    $ make > /dev/null
    $ ctest .
    Test project ./bld
    Start 1: tests/test_fails_1.f90
    1/2 Test #1: tests/test_fails_1.f90 ...........***Failed    0.00 sec
    Start 2: tests/test_succeeds_1.f90
    2/2 Test #2: tests/test_succeeds_1.f90 ........   Passed    0.00 sec

    50% tests passed, 1 tests failed out of 2

    Total Test time (real) =   0.01 sec

    The following tests FAILED:
              1 - tests/test_fails_1.f90 (Failed)
    Errors while running CTest
    $

This is the output that is expected: one test succeeds and one test fails.

Links
-----

- `Sample code on github <https://github.com/pyohannes/ctest-fortran-unittest>`_
- `Fortran name mangling <https://en.wikipedia.org/wiki/Name_mangling#Fortran>`_
- `The CMake function create_test_sourcelist <https://cmake.org/cmake/help/v3.5/command/create_test_sourcelist.html>`_
