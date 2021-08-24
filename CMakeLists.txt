# ---------------------------------------------------------------------------
# Project name prefix

string(TOUPPER "${CMAKE_PROJECT_NAME}" P)
string(REPLACE "-" "_" P "${P}")
set(PN "${CMAKE_PROJECT_DESCRIPTION}")

# ---------------------------------------------------------------------------
# Is this the master project?

set(${P}_MASTER_PROJECT OFF)
if(${CMAKE_PROJECT_NAME} STREQUAL ${PROJECT_NAME})
  set(${P}_MASTER_PROJECT ON)
endif()

# ---------------------------------------------------------------------------
# Only run the rest of this file a single time

if (MTS_CMAKE_DEFAULTS)
  unset(PN)
  unset(P)
  return()
endif()

set(MTS_CMAKE_DEFAULTS TRUE)

# ---------------------------------------------------------------------------
# 32 bit architectures unsupported

if(CMAKE_SIZEOF_VOID_P STREQUAL "4")
  message(FATAL_ERROR "${PN}: project requires a 64 bit architecture!")
endif()

# ---------------------------------------------------------------------------
# Do a release build if nothing was specified

if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "${PN}: setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

# ---------------------------------------------------------------------------

# Compile for the native system architecture (assumes AVX2 on windows). When
# enoki is compiled as part of a PyPI package via scikit-build, adopt a lower
# target architecture (Ivy Bridge, which has the AVX and F16C extensions). An
# M1-like architecture is assumed for arm64 builds on macOS.

if (MSVC)
  if (SKBUILD)
    # Reasonably portable binaries for PyPI
    set(${P}_NATIVE_FLAGS_DEFAULT "/arch:AVX")
  else()
    set(${P}_NATIVE_FLAGS_DEFAULT "/arch:AVX2")
  endif()
elseif (APPLE AND CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64")
  set(${P}_NATIVE_FLAGS_DEFAULT "-mcpu=apple-a12")
else()
  if (SKBUILD)
    # Reasonably portable binaries for PyPI
    set(${P}_NATIVE_FLAGS_DEFAULT "-march=ivybridge")
  else()
    set(${P}_NATIVE_FLAGS_DEFAULT "-march=native")
  endif()
endif()

if (SKBUILD)
  message(STATUS "${PN}: setting portable compilation defaults for scikit-build.")
else()
  message(STATUS "${PN}: targeting the native CPU architecture (specify ${P}_NATIVE_FLAGS to change this).")
endif()

set(${P}_NATIVE_FLAGS ${${P}_NATIVE_FLAGS_DEFAULT} CACHE STRING
    "Compilation flags used to target the host processor architecture.")

add_compile_options(${${P}_NATIVE_FLAGS})
unset(${P}_NATIVE_FLAGS_DEFAULT)

# ---------------------------------------------------------------------------
# Prefer libc++ in conjunction with Clang

include(CheckCXXSourceRuns)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

macro(CHECK_CXX_COMPILER_AND_LINKER_FLAGS _RESULT _CXX_FLAGS _LINKER_FLAGS)
  set(CMAKE_REQUIRED_FLAGS ${_CXX_FLAGS})
  set(CMAKE_REQUIRED_LIBRARIES ${_LINKER_FLAGS})
  set(CMAKE_REQUIRED_QUIET TRUE)
  check_cxx_source_runs("#include <iostream>\nint main(int argc, char **argv) { std::cout << \"test\"; return 0; }"
                        ${_RESULT})
  set(CMAKE_REQUIRED_FLAGS "")
  set(CMAKE_REQUIRED_LIBRARIES "")
endmacro()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND NOT CMAKE_CXX_FLAGS MATCHES "-stdlib=libc\\+\\+")
  CHECK_CXX_COMPILER_AND_LINKER_FLAGS(HAS_LIBCPP "-stdlib=libc++" "-stdlib=libc++")
  if (APPLE OR HAS_LIBCPP)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++ -D_LIBCPP_VERSION")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -stdlib=libc++")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -stdlib=libc++")
    message(STATUS "${PN}: using libc++.")
  else()
    CHECK_CXX_COMPILER_AND_LINKER_FLAGS(HAS_LIBCPP_AND_CPPABI "-stdlib=libc++" "-stdlib=libc++ -lc++abi")
    if (HAS_LIBCPP_AND_CPPABI)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++ -D_LIBCPP_VERSION")
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -stdlib=libc++ -lc++abi")
      set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -stdlib=libc++ -lc++abi")
      message(STATUS "${PN}: using libc++ and libc++abi.")
    else()
      message(FATAL_ERROR "When Clang is used to compile ${PN}, libc++ "
          "must be available -- GCC's libstdc++ is not supported! (please install "
          "the libc++ development headers, provided e.g. by the packages "
          "'libc++-dev' and 'libc++abi-dev' on Debian/Ubuntu).")
    endif()
  endif()
endif()

# ---------------------------------------------------------------------------
# Symbol visibility = hidden by default

set(CMAKE_CXX_VISIBILITY_PRESET hidden)

# ---------------------------------------------------------------------------
# Compile with a few more compiler warnings turned on

if (MSVC)
  if (CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
    string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  else()
    add_compile_options(/W4)
  endif()
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  add_compile_options(-Wall -Wextra)
endif()

# ---------------------------------------------------------------------------
# Sane defaults for MSVC

if (MSVC)
   add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_DEPRECATE -DNOMINMAX)
endif()

# ---------------------------------------------------------------------------
# Find Python if the parent project includes Python bindings

if (${P}_ENABLE_PYTHON)
  if (NOT TARGET pybind11::module)
    if (SKBUILD)
      execute_process(
        COMMAND
        "${PYTHON_EXECUTABLE}" -c
        "import pybind11; print(pybind11.get_cmake_dir())"
        OUTPUT_VARIABLE _tmp_dir
        OUTPUT_STRIP_TRAILING_WHITESPACE COMMAND_ECHO STDOUT)
        list(APPEND CMAKE_PREFIX_PATH "${_tmp_dir}")
      find_package(pybind11 CONFIG REQUIRED)
    else()
      set(Python_FIND_FRAMEWORK LAST) # Prefer Brew/Conda to Apple framework python
      find_package(Python COMPONENTS Interpreter Development REQUIRED)
      set(${P}_PYBIND11_DIR "${CMAKE_CURRENT_SOURCE_DIR}/ext/pybind11"
        CACHE STRING "Path containing the 'pybind11' library used to compile ${PN}.")
      add_subdirectory(${${P}_PYBIND11_DIR} pybind11)
      mark_as_advanced(
        USE_PYTHON_INCLUDE_DIR PYBIND11_INSTALL PYBIND11_TEST PYBIND11_NOPYTHON
        PYBIND11_FINDPYTHON PYBIND11_PYTHON_VERSION PYBIND11_WERROR
        ${P}_PYBIND11_DIR)
      endif()
  endif()
endif()

# ---------------------------------------------------------------------------
# Clear temporary variables
unset(PN)
unset(P)
