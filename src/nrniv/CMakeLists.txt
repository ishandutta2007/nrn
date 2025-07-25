# =============================================================================
# Build nrniv binary and corresponding library
# =============================================================================
include(${CODING_CONV_CMAKE}/build-time-copy.cmake)

# Add directory-level default compiler flags -- these should be added to all NEURON targets, but not
# targets from included projects like CoreNEURON and NMODL
add_compile_options(${NRN_COMPILE_FLAGS})
add_compile_definitions(${NRN_COMPILE_DEFS})
add_link_options(${NRN_LINK_FLAGS})

if(NRN_AVOID_ABSOLUTE_PATHS)
  add_compile_definitions(NRN_AVOID_ABSOLUTE_PATHS)
endif()

# =============================================================================
# Build modlunit : Mod file units checker
# =============================================================================
set(BISON_FLEX_WORKING_DIR "${CMAKE_BINARY_DIR}")
set(NRN_MODLUNIT_GEN "${CMAKE_CURRENT_BINARY_DIR}/modlunit_generated")
file(MAKE_DIRECTORY "${NRN_MODLUNIT_GEN}")
file(RELATIVE_PATH NRN_MODLUNIT_GEN_REL "${BISON_FLEX_WORKING_DIR}" "${NRN_MODLUNIT_GEN}")
file(RELATIVE_PATH NRN_MODLUNIT_SRC_REL "${BISON_FLEX_WORKING_DIR}" "${NRN_MODLUNIT_SRC_DIR}")
# Run flex and bison with relative paths, so absolute paths are not present in the generated source
# file because of include, __FILE__ and so on. This improves ccache performance. To generate code
# coverage correctly, the *relative* paths that are present need to be with respect to the build
# directory.
add_custom_command(
  OUTPUT "${NRN_MODLUNIT_GEN}/lex.cpp"
  WORKING_DIRECTORY "${BISON_FLEX_WORKING_DIR}"
  COMMAND "${FLEX_EXECUTABLE}" ARGS -o "${NRN_MODLUNIT_GEN_REL}/lex.cpp"
          "${NRN_MODLUNIT_SRC_REL}/lex.lpp"
  DEPENDS "${NRN_MODLUNIT_SRC_DIR}/lex.lpp"
  COMMENT "[FLEX][modlunitlexer] Building scanner with flex ${FLEX_VERSION}")
add_custom_command(
  OUTPUT "${NRN_MODLUNIT_GEN}/parse1.hpp" "${NRN_MODLUNIT_GEN}/parse1.cpp"
  WORKING_DIRECTORY "${BISON_FLEX_WORKING_DIR}"
  COMMAND "${BISON_EXECUTABLE}" ARGS "--defines=${NRN_MODLUNIT_GEN_REL}/parse1.hpp" -o
          "${NRN_MODLUNIT_GEN_REL}/parse1.cpp" "${NRN_MODLUNIT_SRC_REL}/parse1.ypp"
  DEPENDS "${NRN_MODLUNIT_SRC_DIR}/parse1.ypp"
  COMMENT "[BISON][modlunitparser] Building parser with bison ${BISON_VERSION}")

add_custom_target(generated_source_files true)
add_custom_target(modlunit_generated_files DEPENDS "${NRN_MODLUNIT_GEN}/lex.cpp"
                                                   "${NRN_MODLUNIT_GEN}/parse1.cpp")
add_dependencies(generated_source_files modlunit_generated_files)

add_executable(modlunit ${NRN_MODLUNIT_SRC_FILES} "${NRN_MODLUNIT_GEN}/lex.cpp"
                        "${NRN_MODLUNIT_GEN}/parse1.cpp")
add_dependencies(modlunit modlunit_generated_files)
target_include_directories(modlunit PRIVATE "${PROJECT_BINARY_DIR}/share/nrn/lib")

target_compile_definitions(modlunit PRIVATE NRNUNIT=1)
cpp_cc_configure_sanitizers(TARGET modlunit)
# Generated .cpp needs to find source-directory .hpp and vice versa.
target_include_directories(modlunit PRIVATE "${NRN_MODLUNIT_SRC_DIR}" "${NRN_MODLUNIT_GEN}")

if(NRN_NOCMODL_CXX_FLAGS)
  target_compile_options(modlunit PRIVATE ${NRN_NOCMODL_CXX_FLAGS})
endif()

# =============================================================================
# Build nocmodl : source-to-source compiler for NOCMODL
# =============================================================================
set(NRN_NOCMODL_GEN "${CMAKE_CURRENT_BINARY_DIR}/nocmodl_generated")
file(MAKE_DIRECTORY "${NRN_NOCMODL_GEN}")
file(RELATIVE_PATH NRN_NOCMODL_GEN_REL "${BISON_FLEX_WORKING_DIR}" "${NRN_NOCMODL_GEN}")
file(RELATIVE_PATH NRN_NOCMODL_SRC_REL "${BISON_FLEX_WORKING_DIR}" "${NRN_NOCMODL_SRC_DIR}")
add_custom_command(
  OUTPUT "${NRN_NOCMODL_GEN}/lex.cpp"
  WORKING_DIRECTORY "${BISON_FLEX_WORKING_DIR}"
  COMMAND "${FLEX_EXECUTABLE}" ARGS -o "${NRN_NOCMODL_GEN_REL}/lex.cpp"
          "${NRN_NOCMODL_SRC_REL}/lex.lpp"
  DEPENDS "${NRN_NOCMODL_SRC_DIR}/lex.lpp"
  COMMENT "[FLEX][nocmodllexer] Building scanner with flex ${FLEX_VERSION}")
add_custom_command(
  OUTPUT "${NRN_NOCMODL_GEN}/parse1.hpp" "${NRN_NOCMODL_GEN}/parse1.cpp"
  WORKING_DIRECTORY "${BISON_FLEX_WORKING_DIR}"
  COMMAND "${BISON_EXECUTABLE}" ARGS "--defines=${NRN_NOCMODL_GEN_REL}/parse1.hpp" -o
          "${NRN_NOCMODL_GEN_REL}/parse1.cpp" "${NRN_NOCMODL_SRC_REL}/parse1.ypp"
  DEPENDS "${NRN_NOCMODL_SRC_DIR}/parse1.ypp"
  COMMENT "[BISON][nocmodlparser] Building parser with bison ${BISON_VERSION}")
add_custom_command(
  OUTPUT "${NRN_NOCMODL_GEN}/diffeq.hpp" "${NRN_NOCMODL_GEN}/diffeq.cpp"
  WORKING_DIRECTORY "${BISON_FLEX_WORKING_DIR}"
  COMMAND "${BISON_EXECUTABLE}" ARGS "--defines=${NRN_NOCMODL_GEN_REL}/diffeq.hpp" -o
          "${NRN_NOCMODL_GEN_REL}/diffeq.cpp" "${NRN_NOCMODL_SRC_REL}/diffeq.ypp"
  DEPENDS "${NRN_NOCMODL_SRC_DIR}/diffeq.ypp"
  COMMENT "[BISON][nocmodlparser] Building diffeq with bison ${BISON_VERSION}")

add_custom_target(
  nocmodl_generated_files
  DEPENDS "${NRN_NOCMODL_GEN}/lex.cpp" "${NRN_NOCMODL_GEN}/parse1.hpp"
          "${NRN_NOCMODL_GEN}/parse1.cpp" "${NRN_NOCMODL_GEN}/diffeq.hpp"
          "${NRN_NOCMODL_GEN}/diffeq.cpp")
add_dependencies(generated_source_files nocmodl_generated_files)

add_executable(nocmodl ${NRN_NOCMODL_SRC_FILES} "${NRN_NOCMODL_GEN}/lex.cpp"
                       "${NRN_NOCMODL_GEN}/parse1.cpp" "${NRN_NOCMODL_GEN}/diffeq.cpp")
target_include_directories(nocmodl PRIVATE "${PROJECT_BINARY_DIR}/share/nrn/lib")
cpp_cc_configure_sanitizers(TARGET nocmodl)
add_dependencies(nocmodl nocmodl_generated_files)
target_compile_definitions(nocmodl PRIVATE COMPILE_DEFINITIONS NMODL=1 CVODE=1)
# Otherwise the generated code in the binary directory does not find headers in the modlunit source
# directory and the source files in the source directory do not find generated headers in the binary
# directory. TODO: (see also coreneuron) fix adding a dependency on CLI11::CLI11 when CLI11 is a
# submodule. Right now this doesn't work because the CLI11 targets are not exported/installed but
# coreneuron-core is.
get_target_property(CLI11_HEADER_DIRECTORY CLI11::CLI11 INTERFACE_INCLUDE_DIRECTORIES)
target_include_directories(nocmodl PRIVATE "${NRN_NOCMODL_GEN}" "${NRN_NOCMODL_SRC_DIR}"
                                           "${CLI11_HEADER_DIRECTORY}")
if(NRN_NOCMODL_CXX_FLAGS)
  target_compile_options(nocmodl PRIVATE ${NRN_NOCMODL_CXX_FLAGS})
endif()

# CLI11-included headers have some warnings on NVHPC
if(NRN_NOCMODL_COMPILER_WARNING_SUPPRESSIONS)
  target_compile_options(nocmodl PRIVATE ${NRN_NOCMODL_COMPILER_WARNING_SUPPRESSIONS})
endif()

# =============================================================================
# Translate all MOD files to C and mark them generated
# =============================================================================
foreach(modfile ${NRN_MODFILE_BASE_NAMES})
  set(modfile_compat OFF)
  if((NRN_ENABLE_CORENEURON OR NRN_ENABLE_MOD_COMPATIBILITY) AND modfile MATCHES "hh$")
    set(modfile_compat ON)
  endif()
  nocmodl_mod_to_cpp(${modfile} ${modfile_compat})
  list(APPEND NRN_MODFILE_CPP ${PROJECT_BINARY_DIR}/${modfile}.cpp)
endforeach()
set_property(
  SOURCE ${NRN_MODFILE_CPP}
  APPEND
  PROPERTY COMPILE_OPTIONS ${NRN_EXTRA_MECH_CXX_FLAGS})
# we are basically emulating nrnivmodl with CMake here. nrnivmodl uses headers from the include
# directory in the build or install directory.
set_property(
  SOURCE ${NRN_MODFILE_CPP}
  APPEND
  PROPERTY INCLUDE_DIRECTORIES ${PROJECT_BINARY_DIR}/include)
set_source_files_properties(${NRN_MODFILE_CPP} PROPERTIES GENERATED TRUE)

# =============================================================================
# Bison parser for HOC interpreter
# =============================================================================
set(NRN_OC_GEN "${CMAKE_CURRENT_BINARY_DIR}/oc_generated")
file(MAKE_DIRECTORY "${NRN_OC_GEN}")
file(RELATIVE_PATH NRN_OC_GEN_REL "${BISON_FLEX_WORKING_DIR}" "${NRN_OC_GEN}")
file(RELATIVE_PATH NRN_OC_SRC_REL "${BISON_FLEX_WORKING_DIR}" "${NRN_OC_SRC_DIR}")
add_custom_command(
  OUTPUT "${NRN_OC_GEN}/parse.hpp" "${NRN_OC_GEN}/parse.cpp"
  WORKING_DIRECTORY "${BISON_FLEX_WORKING_DIR}"
  COMMAND "${BISON_EXECUTABLE}" ARGS "--defines=${NRN_OC_GEN_REL}/parse.hpp" -o
          "${NRN_OC_GEN_REL}/parse.cpp" "${NRN_OC_SRC_REL}/parse.ypp"
  DEPENDS "${NRN_OC_SRC_DIR}/parse.ypp"
  COMMENT "[BISON][ocparser] Building parser with bison ${BISON_VERSION}")
add_custom_target(oc_generated_files DEPENDS "${NRN_OC_GEN}/parse.hpp" "${NRN_OC_GEN}/parse.cpp")
add_dependencies(generated_source_files oc_generated_files)
list(APPEND NRN_OC_SRC_FILES "${NRN_OC_GEN}/parse.cpp")
set(NRN_OC_GENERATED_SOURCES
    "${NRN_OC_GEN}"
    PARENT_SCOPE)

# =============================================================================
# Source code lists
# =============================================================================
set(NRN_NRNIV_LIB_SRC_FILES
    ${NRN_IVOC_SRC_FILES}
    ${NRN_NODEORDEROPTIM_SRC_FILES}
    ${NRN_NRNCVODE_SRC_FILES}
    ${NRN_NRNIV_SRC_FILES}
    ${NRN_NRNOC_SRC_FILES}
    ${NRN_OC_SRC_FILES}
    ${NRN_MODFILE_CPP}
    ${NRN_SCOPMATH_SRC_FILES}
    ${NRN_SPARSE_SRC_FILES}
    ${NRN_SUNDIALS_SRC_FILES}
    ${NRN_PARALLEL_SRC_FILES})

if(NRN_ENABLE_MPI_DYNAMIC)
  list(APPEND NRN_NRNIV_LIB_SRC_FILES ${PROJECT_SOURCE_DIR}/src/nrnmpi/nrnmpi_dynam.cpp)
else()
  list(APPEND NRN_NRNIV_LIB_SRC_FILES ${NRN_NRNMPI_SRC_FILES})
endif()

if(MINGW)
  list(APPEND NRN_NRNIV_LIB_SRC_FILES ${PROJECT_SOURCE_DIR}/src/mswin/dlfcn.c)
endif()

if(NOT NRN_ENABLE_INTERVIEWS)
  list(APPEND NRN_NRNIV_LIB_SRC_FILES ${NRN_IVOS_SRC_FILES})
endif()

if(NRN_ENABLE_MUSIC)
  if(NRN_ENABLE_MPI_DYNAMIC)
    list(APPEND NRN_NRNIV_LIB_SRC_FILES ${PROJECT_SOURCE_DIR}/src/neuronmusic/nrnmusic_dynam.cpp)
    set_property(
      SOURCE ${NRN_NRNIV_LIB_SRC_FILES} ${NRN_NRNMPI_SRC_FILES}
      APPEND
      PROPERTY COMPILE_DEFINITIONS NRNMUSIC_DYNAMIC=1)
  else()
    list(APPEND NRN_NRNIV_LIB_SRC_FILES ${NRN_MUSIC_SRC_FILES})
  endif()
endif()

# =============================================================================
# Include directories for #include <../../nrnconf.h>
# =============================================================================
set(NRN_INCLUDE_DIRS
    ${NRN_IVOC_SRC_DIR}
    ${NRN_NRNCVODE_SRC_DIR}
    ${NRN_NRNIV_SRC_DIR}
    ${NRN_NRNOC_SRC_DIR}
    ${NRN_OC_SRC_DIR}
    ${PROJECT_BINARY_DIR}
    ${PROJECT_BINARY_DIR}/src/nrncvode
    ${PROJECT_BINARY_DIR}/src/nrnmpi
    ${PROJECT_BINARY_DIR}/src/nrnoc
    ${PROJECT_BINARY_DIR}/src/nrnpython
    ${PROJECT_BINARY_DIR}/src/oc
    ${PROJECT_BINARY_DIR}/src/parallel
    ${PROJECT_BINARY_DIR}/src/sundials
    ${PROJECT_BINARY_DIR}/src/sundials/shared
    ${PROJECT_SOURCE_DIR}/src
    ${PROJECT_SOURCE_DIR}/src/gnu
    ${PROJECT_SOURCE_DIR}/src/nrncvode
    ${PROJECT_SOURCE_DIR}/src/nrnmpi
    ${PROJECT_SOURCE_DIR}/src/nrnpython
    ${PROJECT_SOURCE_DIR}/src/parallel
    ${PROJECT_SOURCE_DIR}/src/scopmath
    ${PROJECT_SOURCE_DIR}/src/sparse
    ${PROJECT_SOURCE_DIR}/src/sparse13
    ${PROJECT_SOURCE_DIR}/src/sundials
    ${PROJECT_SOURCE_DIR}/src/sundials/cvodes
    ${PROJECT_SOURCE_DIR}/src/sundials/ida
    ${PROJECT_SOURCE_DIR}/src/sundials/shared)

if(NRN_ENABLE_MUSIC)
  list(APPEND NRN_INCLUDE_DIRS ${NRN_MUSIC_SRC_DIR})
endif()

set(NRN_INCLUDE_DIRS
    ${NRN_INCLUDE_DIRS}
    PARENT_SCOPE)

# =============================================================================
# Helper commands : generate various headers
# =============================================================================

# avoid error with nvector_serial.cpp for #include <../../../nrnconf.h>
file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/src/sundials/shared)

# generate hocusr.h
add_custom_command(
  OUTPUT ${PROJECT_BINARY_DIR}/src/oc/hocusr.h
  COMMAND ${CMAKE_C_COMPILER} -E -I${NRN_NRNOC_SRC_DIR} -I${NRN_OC_SRC_DIR}
          ${NRN_NRNOC_SRC_DIR}/neuron.h > neuron.tmp1
  COMMAND sed "/^#/d" neuron.tmp1 > neuron.tmp2
  COMMAND ${NRN_DEFAULT_PYTHON_EXECUTABLE} ${NRN_OC_SRC_DIR}/mk_hocusr_h.py < neuron.tmp2 >
          ${PROJECT_BINARY_DIR}/src/oc/hocusr.h
  DEPENDS ${NRN_NRNOC_SRC_DIR}/neuron.h ${NRN_OC_SRC_DIR}/mk_hocusr_h.py)
add_custom_target(generate_hocusr_header DEPENDS "${PROJECT_BINARY_DIR}/src/oc/hocusr.h")
add_dependencies(generated_source_files generate_hocusr_header)

# header for dynamic mpi
if(NRN_ENABLE_MPI_DYNAMIC)
  file(MAKE_DIRECTORY "${PROJECT_BINARY_DIR}/src/nrnmpi")
  add_custom_command(
    OUTPUT ${NRNMPI_DYNAMIC_INCLUDE_FILE}
    COMMAND sh mkdynam.sh "${PROJECT_BINARY_DIR}/src/nrnmpi"
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/src/nrnmpi
    DEPENDS ${PROJECT_SOURCE_DIR}/src/nrnmpi/mkdynam.sh
            ${PROJECT_SOURCE_DIR}/src/nrnmpi/nrnmpidec.h)
endif()

# =============================================================================
# Various macro definitions for compiling different files
# =============================================================================
set_property(
  SOURCE ${NRN_OC_SRC_FILES} ${NRN_SPARSE_SRC_FILES}
  APPEND
  PROPERTY COMPILE_DEFINITIONS OOP=1 HOC=1 INTERVIEWS=1)

set_property(
  SOURCE ${NRN_IVOC_SRC_FILES}
  APPEND
  PROPERTY COMPILE_DEFINITIONS USEMATRIX=1 USEGNU=1 USEBBS=1)

set_property(
  SOURCE ${NRN_NRNIV_SRC_FILES} ${NRN_NRNCVODE_SRC_FILES}
  APPEND
  PROPERTY COMPILE_DEFINITIONS USEMATRIX=1 USECVODE=1 USEBBS=1 OOP=1)

set_property(
  SOURCE ${NRN_SCOPMATH_SRC_FILES}
  APPEND
  PROPERTY COMPILE_DEFINITIONS HOC=1)

set_property(
  SOURCE ${NRN_SCOPMATH_SRC_FILES}
  APPEND
  PROPERTY COMPILE_FLAGS "-I${PROJECT_SOURCE_DIR}/src/scopmath")

if(DEF_RL_GETC_FUNCTION)
  set_property(
    SOURCE ${NRN_OC_SRC_DIR}/hoc.cpp
    APPEND
    PROPERTY COMPILE_DEFINITIONS ${DEF_RL_GETC_FUNCTION})
endif()

if(NRN_USE_BACKWARD)
  set_property(
    SOURCE ${NRN_OC_SRC_DIR}/hoc.cpp
    APPEND
    PROPERTY COMPILE_DEFINITIONS USE_BACKWARD=1)
  set_property(
    SOURCE ${NRN_NRNIV_SRC_DIR}/backtrace_utils.cpp
    APPEND
    PROPERTY COMPILE_DEFINITIONS USE_BACKWARD=1)
endif()

if(NRN_HAVE_NVHPC_COMPILER)
  # For NVHPC we will rely on FE exceptions as opposed to errno in order to make use of faster
  # builtins. One caveat is that if we use an optimization level greater than -O1, the FE exception
  # is not raised. See https://github.com/neuronsimulator/nrn/pull/1930
  set(NVHPC_MATH_COMPILE_OPTIONS "-O1")
  if(${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER_EQUAL 23.1)
    list(APPEND NVHPC_MATH_COMPILE_OPTIONS "-Kieee")
  endif()
  set_source_files_properties(
    ${PROJECT_SOURCE_DIR}/src/oc/math.cpp
    PROPERTIES COMPILE_DEFINITIONS NRN_CHECK_FE_EXCEPTIONS=1 COMPILE_OPTIONS
                                                             "${NVHPC_MATH_COMPILE_OPTIONS}")
  # Versions of nvc++ around ~22.5 up to at least 23.1 have problems with this file see
  # https://github.com/BlueBrain/CoreNeuron/issues/888
  set_source_files_properties(${PROJECT_SOURCE_DIR}/src/ivoc/ivocvect.cpp PROPERTIES COMPILE_OPTIONS
                                                                                     "-O1")
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "IntelLLVM")
  # Tested with icpx 2022.2.1:
  #
  # * With -fp-model=fast (the default...), icpx does not set the C++11 macro math_errhandling.
  # * With -fp-model=fast and anything but -O0, icpx does not set floating point exception bits.
  # * With -fp-model={precise,strict}, floating point exceptions are set at all optimisation levels.
  # * With -fp-model=precise and anything but -O0, -fno-math-errno also disables exceptions.
  # * When defined, math_errhandling always contains both MATH_ERRNO and MATH_ERREXCEPT.
  #
  # Based on all of this, forcing strict mode and forcing floating point exceptions to be used (with
  # NRN_CHECK_FE_EXCEPTIONS) seems that it should work at all optimisation levels. We might as well
  # also pass -fno-math-errno because we will not be checking errno.
  set_property(
    SOURCE ${PROJECT_SOURCE_DIR}/src/oc/math.cpp
    APPEND
    PROPERTY COMPILE_DEFINITIONS NRN_CHECK_FE_EXCEPTIONS)
  set_property(
    SOURCE ${PROJECT_SOURCE_DIR}/src/oc/math.cpp
    APPEND
    PROPERTY COMPILE_OPTIONS "-fp-model=strict" "-fno-math-errno")
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "Intel")
  # When optimisation is enabled then icpc apparently does not set errno
  set_property(
    SOURCE ${PROJECT_SOURCE_DIR}/src/oc/math.cpp
    APPEND
    PROPERTY COMPILE_OPTIONS -fmath-errno -fno-builtin)
  # At -O0 then the Intel compiler generates extremely slow code for this file
  set_property(
    SOURCE ${PROJECT_SOURCE_DIR}/src/oc/code.cpp
    APPEND
    PROPERTY COMPILE_OPTIONS -O1)
endif()

if(NRN_ENABLE_MPI_DYNAMIC)
  set_source_files_properties(${PROJECT_SOURCE_DIR}/src/nrnmpi/nrnmpi_dynam.cpp
                              PROPERTIES OBJECT_DEPENDS ${NRNMPI_DYNAMIC_INCLUDE_FILE})
  set_source_files_properties(${PROJECT_SOURCE_DIR}/src/nrnmpi/nrnmpi.cpp
                              PROPERTIES OBJECT_DEPENDS ${NRNMPI_DYNAMIC_INCLUDE_FILE})
  set_source_files_properties(${PROJECT_SOURCE_DIR}/src/nrnmpi/memory_usage.cpp
                              PROPERTIES OBJECT_DEPENDS ${NRNMPI_DYNAMIC_INCLUDE_FILE})
  set_source_files_properties(${PROJECT_SOURCE_DIR}/src/nrnmpi/bbsmpipack.cpp
                              PROPERTIES OBJECT_DEPENDS ${NRNMPI_DYNAMIC_INCLUDE_FILE})
endif()

if(NRN_ENABLE_MUSIC)
  set_source_files_properties(${PROJECT_SOURCE_DIR}/src/neuronmusic/nrnmusic.cpp
                              PROPERTIES INCLUDE_DIRECTORIES ${MUSIC_INCDIR})
endif()

# Maybe NRN_COVERAGE_FILES mentions a file compiled by this CMakeLists.txt
nrn_enable_coverage_files()

# =============================================================================
# All source directories to include
# =============================================================================
include_directories(${NRN_INCLUDE_DIRS})

# =============================================================================
# All source directories to include
# =============================================================================
add_library(nrniv_lib ${NRN_LIBRARY_TYPE} ${NRN_NRNIV_LIB_SRC_FILES})
add_dependencies(nrniv_lib generated_source_files)
add_cpp_git_information(nrniv_lib PRIVATE)
target_link_libraries(nrniv_lib nrngnu)
target_link_libraries(nrniv_lib sparse13)
target_link_libraries(nrniv_lib fmt::fmt)
target_include_directories(nrniv_lib SYSTEM PUBLIC ${PROJECT_SOURCE_DIR}/${NRN_3RDPARTY_DIR}/eigen)
cpp_cc_configure_sanitizers(TARGET nrniv_lib)
# Source-directory .cpp needs to find generated .hpp.
target_include_directories(nrniv_lib PUBLIC "${NRN_OC_GEN}")
if(NRN_ENABLE_PYTHON AND NOT NRN_ENABLE_PYTHON_DYNAMIC)
  target_link_libraries(nrniv_lib nrnpython)
endif()
if(NRN_ENABLE_THREADS)
  target_link_libraries(nrniv_lib Threads::Threads)
endif()

if(${NRN_ENABLE_DIGEST})
  if(NRN_MACOS_BUILD)
    # where to get openssl/sha.h after brew install openssl
    set_property(
      SOURCE ${NRN_OC_SRC_DIR}/debug.cpp
      APPEND
      PROPERTY INCLUDE_DIRECTORIES /opt/homebrew/Cellar/openssl@3/3.1.0/include)
    find_library(LIB_CRYPTO crypto PATHS /opt/homebrew/Cellar/openssl@3/3.1.0/lib REQUIRED)
    target_link_libraries(nrniv_lib ${LIB_CRYPTO})
  else()
    target_link_libraries(nrniv_lib crypto)
  endif()
endif()

if(${NRN_ENABLE_ARCH_INDEP_EXP_POW})
  find_library(LIB_MPFR mpfr REQUIRED)
  target_link_libraries(nrniv_lib ${LIB_MPFR})
endif()

if(NRN_WINDOWS_BUILD)
  target_link_libraries(nrniv_lib ${TERMCAP_LIBRARIES} ${Readline_LIBRARY})
else()
  if(READLINE_FOUND)
    target_link_libraries(nrniv_lib ${Readline_LIBRARY})
  else()
    target_link_libraries(nrniv_lib readline)
  endif()

  if(CURSES_FOUND)
    target_link_libraries(nrniv_lib ${CURSES_LIBRARIES})
  elseif(TERMCAP_FOUND)
    target_link_libraries(nrniv_lib ${TERMCAP_LIBRARIES})
  endif()
endif()

if(NRN_ENABLE_MUSIC AND NOT NRN_ENABLE_MPI_DYNAMIC)
  target_link_libraries(nrniv_lib ${MUSIC_LIBRARY})
endif()

if(NRN_ENABLE_PROFILING)
  target_link_libraries(nrniv_lib ${likwid_LIBRARIES} ${CALIPER_LIB} ${LIKWID_LIB})
endif()

set_property(TARGET nrniv_lib PROPERTY OUTPUT_NAME nrniv)

# =============================================================================
# Link with backward-cpp if enabled
# =============================================================================
if(NRN_USE_BACKWARD)
  add_backward(nrniv_lib)
endif()

# =============================================================================
# Link with all libraries
# =============================================================================
if(NRN_ENABLE_MPI)
  if(NRN_ENABLE_MPI_DYNAMIC)
    list(LENGTH NRN_MPI_LIBNAME_LIST _num_mpi)
    math(EXPR num_mpi "${_num_mpi} - 1")
    foreach(val RANGE ${num_mpi})
      list(GET NRN_MPI_INCLUDE_LIST ${val} include)
      list(GET NRN_MPI_LIBNAME_LIST ${val} libname)

      add_library(${libname}_lib SHARED ${NRN_NRNMPI_SRC_FILES})
      target_link_libraries(${libname}_lib fmt::fmt)
      target_include_directories(${libname}_lib BEFORE PUBLIC ${include})
      # Note that we do not link here to libmpi. That is dlopen first.
      if(MINGW) # type msmpi only
        add_dependencies(${libname}_lib nrniv_lib)
        target_link_libraries(${libname}_lib ${MPI_C_LIBRARIES})
        target_link_libraries(${libname}_lib nrniv_lib)
      endif()
      set_property(TARGET ${libname}_lib PROPERTY OUTPUT_NAME ${libname})
      install(TARGETS ${libname}_lib DESTINATION ${NRN_INSTALL_SHARE_LIB_DIR})
    endforeach(val)
    if(NRN_ENABLE_MUSIC)
      # libnrnmusic does not depend explicitly on MPI so don't need mpi version specific versions.
      set(libnrnmusic "nrnmusic")
      set(NRNMUSIC_DYNAM_LIB
          "'${libnrnmusic}'"
          PARENT_SCOPE)
      add_library(${libnrnmusic}_lib SHARED ${NRN_MUSIC_SRC_FILES})
      target_include_directories(${libnrnmusic}_lib BEFORE PUBLIC ${include})
      add_dependencies(${libnrnmusic}_lib nrniv_lib ${libname}_lib)
      target_link_libraries(${libnrnmusic}_lib nrniv_lib ${libname}_lib)
      set_property(TARGET ${libnrnmusic}_lib PROPERTY OUTPUT_NAME ${libnrnmusic})
      install(TARGETS ${libnrnmusic}_lib DESTINATION ${NRN_INSTALL_SHARE_LIB_DIR})
    endif()
  else()
    target_link_libraries(nrniv_lib ${MPI_C_LIBRARIES})
    target_include_directories(nrniv_lib PUBLIC ${MPI_INCLUDE_PATH})
  endif()
endif()

if(NRN_ENABLE_INTERVIEWS)
  include_directories(${IV_INCLUDE_DIR})
  target_link_libraries(nrniv_lib interviews)
else()
  target_include_directories(nrniv_lib PUBLIC ${NRN_IVOS_SRC_DIR})
endif()

if(IV_ENABLE_X11_DYNAMIC)
  # ~~~
  # by defining IVX11_DYNAM in ivocmain.cpp, the latter can call ivx11_dyload and
  # if that fails, set hoc_usegui = 0 which avoids all InterViews calls.
  # ~~~
  set_property(
    SOURCE ${PROJECT_SOURCE_DIR}/src/ivoc/ivocmain.cpp ${PROJECT_SOURCE_DIR}/src/pwman/xdep.cpp
           ${PROJECT_SOURCE_DIR}/src/ivoc/xdep.cpp ${PROJECT_SOURCE_DIR}/src/oc/x.cpp
    APPEND
    PROPERTY COMPILE_DEFINITIONS IVX11_DYNAM)
  if((NOT IV_ENABLE_SHARED) AND IV_LIB_DIR)
    # ~~~
    # IV_LIB_DIR is not set when IV is a submodule and not yet installed but
    # libivx11dynam is already in its proper place at POST_BUILD. When
    # IV_ENABLE_SHARED=ON, libivx11dynam is found in IV_LIB_DIR (the location
    # of libinterviews). When OFF, libivx11dynam needs to be copied to the
    # location of libnrniv. The goal is that if libnrniv is loaded, the system
    # can find libivx11dynam independent of CMAKE_INSTALL_PREFIX.
    # ~~~
    set(_suffix ${CMAKE_SHARED_LIBRARY_SUFFIX})
    set(LIBIVX11DYNAM_NAME libivx11dynam${_suffix})
    add_custom_command(
      TARGET nrniv_lib
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy ${IV_LIB_DIR}/${LIBIVX11DYNAM_NAME}
              ${PROJECT_BINARY_DIR}/lib/${LIBIVX11DYNAM_NAME})
  endif()
else()
  target_link_libraries(nrniv_lib ${X11_LIBRARIES})
endif()

# =============================================================================
# Final executable
# =============================================================================
add_executable(nrniv ${NRN_BIN_SRC_FILES})
cpp_cc_configure_sanitizers(TARGET nrniv)
if(NOT "${NRN_LINK_FLAGS_FOR_ENTRY_POINTS}" STREQUAL "")
  target_link_options(nrniv PUBLIC ${NRN_LINK_FLAGS_FOR_ENTRY_POINTS})
endif()
target_link_libraries(nrniv nrniv_lib ${INTERNAL_READLINE} ${CMAKE_DL_LIBS})
if(NRN_ENABLE_THREADS)
  target_link_libraries(nrniv Threads::Threads)
endif()
if(NOT MINGW)
  target_link_libraries(nrniv_lib ${CMAKE_DL_LIBS})
endif()

# TODO: unset in top level CMake is not working
if(NOT NRN_MACOS_BUILD AND READLINE_FOUND)
  target_link_libraries(nrniv ${Readline_LIBRARY})
endif()

# =============================================================================
# Install binary and library targets
# =============================================================================
# ~~~
# classically, the autotools windows version installed dlls in <inst>/bin
# For now, we keep this distinction as it reduces the PATH and is
# expected when ctypes looks for dlls
# ~~~
install(TARGETS nrniv nocmodl modlunit DESTINATION ${NRN_INSTALL_DATA_PREFIX}/bin)
install(TARGETS nrniv_lib DESTINATION ${NRN_INSTALL_SHARE_LIB_DIR})
if(LIBIVX11DYNAM_NAME)
  install(FILES ${PROJECT_BINARY_DIR}/lib/${LIBIVX11DYNAM_NAME}
          DESTINATION ${NRN_INSTALL_DATA_PREFIX}/lib)
endif()

# =============================================================================
# Install / copy cpp and required headers for binary special flavor
# =============================================================================

# For testneuron CTest
cpp_cc_build_time_copy(
  INPUT ${PROJECT_SOURCE_DIR}/src/ivoc/nrnmain.cpp OUTPUT ${CMAKE_BINARY_DIR}/share/nrn/nrnmain.cpp
                                                          TARGET nrnivmodl_nrnmain_cpp)

cpp_cc_build_time_copy(INPUT ${PROJECT_BINARY_DIR}/src/oc/nrnmpiuse.h
                       OUTPUT ${CMAKE_BINARY_DIR}/include/nrnmpiuse.h TARGET nrnivmodl_nrnmpiuse_h)

cpp_cc_build_time_copy(
  INPUT ${PROJECT_BINARY_DIR}/src/nrncvode/nrnneosm.h
  OUTPUT ${CMAKE_BINARY_DIR}/include/nrncvode/nrnneosm.h TARGET nrnivmodl_nrnneosm_h)

cpp_cc_build_time_copy(INPUT ${PROJECT_BINARY_DIR}/nrnconf.h
                       OUTPUT ${CMAKE_BINARY_DIR}/include/nrnconf.h TARGET nrnivmodl_nrnconf_h)

add_custom_target(
  nrnivmodl_dependency ALL
  DEPENDS ${CMAKE_BINARY_DIR}/share/nrn/nrnmain.cpp ${CMAKE_BINARY_DIR}/include/nrnmpiuse.h
          ${CMAKE_BINARY_DIR}/include/nrncvode/nrnneosm.h ${CMAKE_BINARY_DIR}/include/nrnconf.h)

# For the installation
install(FILES ${PROJECT_SOURCE_DIR}/src/ivoc/nrnmain.cpp
        DESTINATION ${NRN_INSTALL_DATA_PREFIX}/share/nrn)
