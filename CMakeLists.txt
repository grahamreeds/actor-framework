cmake_minimum_required(VERSION 3.0 FATAL_ERROR)
project(caf C CXX)

# -- project options -----------------------------------------------------------

option(BUILD_SHARED_LIBS "Build all modules as shared library" ON)

# -- includes ------------------------------------------------------------------

include(CMakePackageConfigHelpers) # For creating .cmake files
include(CheckCSourceCompiles)      # Check whether compiler works
include(CheckCSourceRuns)          # Check whether compiler produces binaries
include(GNUInstallDirs)            # Sets default install paths
include(GenerateExportHeader)      # Auto-generates dllexport macros

# -- project-specific CMake settings -------------------------------------------

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# -- check whether we are running as CMake subdirectory ------------------------

get_directory_property(_parent PARENT_DIRECTORY)
if(_parent)
  set(caf_is_subproject ON)
else()
  set(caf_is_subproject OFF)
endif()
unset(_parent)

# enable tests if not disabled
if(NOT CAF_NO_UNIT_TESTS)
  enable_testing()
  function(add_test_suites executable dir)
    # enumerate all test suites.
    set(suites "")
    foreach(cpp_file ${ARGN})
      file(STRINGS "${dir}/${cpp_file}" contents)
      foreach(line ${contents})
        if ("${line}" MATCHES "SUITE (.+)")
          string(REGEX REPLACE ".*SUITE (.+)" "\\1" suite ${line})
          list(APPEND suites "${suite}")
        endif()
      endforeach()
    endforeach()
    list(REMOVE_DUPLICATES suites)
    list(LENGTH suites num_suites)
    message(STATUS "Found ${num_suites} test suite for ${executable}")
    # creates one CMake test per test suite.
    macro (make_test suite)
      string(REPLACE " " "_" test_name ${suite})
      add_test(NAME ${test_name} COMMAND ${executable} -r300 -n -v5 -s"^${suite}$")
    endmacro ()
    list(LENGTH suites num_suites)
    foreach(suite ${suites})
      make_test("${suite}")
    endforeach ()
  endfunction()
endif()

# -- make sure we have at least C++17 available --------------------------------

if(NOT CMAKE_CROSSCOMPILING)
  # Check whether the user already provided flags that enable C++ >= 17.
  try_compile(caf_has_cxx_17
              "${CMAKE_CURRENT_BINARY_DIR}"
              "${CMAKE_CURRENT_SOURCE_DIR}/cmake/check-compiler-features.cpp")
  # Try enabling C++17 mode if user-provided flags aren't sufficient.
  if(NOT caf_has_cxx_17)
    if(MSVC)
      set(cxx_flag "/std:c++17")
    else()
      if(CMAKE_CXX_COMPILER_ID MATCHES "Clang"
         AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5)
        set(cxx_flag "-std=c++1z")
      else()
        set(cxx_flag "-std=c++17")
      endif()
    endif()
    # Re-run compiler check.
    try_compile(caf_has_cxx_17
                "${CMAKE_CURRENT_BINARY_DIR}"
                "${CMAKE_CURRENT_SOURCE_DIR}/cmake/check-compiler-features.cpp"
                COMPILE_DEFINITIONS "${cxx_flag}"
                OUTPUT_VARIABLE cxx_check_output)
    if(NOT caf_has_cxx_17)
      MESSAGE(FATAL_ERROR "\nFatal error: unable activate C++17 mode!\
                           \nPlease see README.md for supported compilers.\
                           \n\ntry_compile output:\n${cxx_check_output}")
    endif()
    add_compile_options("${cxx_flag}")
  endif()
endif()

set(CMAKE_INSTALL_CMAKEBASEDIR "${CMAKE_INSTALL_LIBDIR}/cmake" CACHE PATH
    "Base directory for installing cmake specific artifacts")
set(INSTALL_CAF_CMAKEDIR "${CMAKE_INSTALL_CMAKEBASEDIR}/caf")

if(BUILD_SHARED_LIBS)
  set(LINK_TYPE "shared")
  set(CMAKE_CXX_VISIBILITY_PRESET hidden)
  set(CMAKE_VISIBILITY_INLINES_HIDDEN yes)
  if(POLICY CMP0063)
    cmake_policy(SET CMP0063 NEW)
  endif()
else()
  set(CAF_STATIC_BUILD yes)
  set(LINK_TYPE "static")
endif()

# Be nice to VIM users and Clang tools.
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

# Silence policy CMP0042 warning by enabling RPATH explicitly.
if(APPLE AND NOT DEFINED CMAKE_MACOSX_RPATH)
  set(CMAKE_MACOSX_RPATH true)
endif()

if(CAF_BUILD_STATIC_RUNTIME)
    set(flags_configs
        CMAKE_CXX_FLAGS
        CMAKE_CXX_FLAGS_DEBUG
        CMAKE_CXX_FLAGS_RELEASE
        CMAKE_CXX_FLAGS_RELWITHDEBINFO
        CMAKE_CXX_FLAGS_MINSIZEREL
        CMAKE_C_FLAGS
        CMAKE_C_FLAGS_DEBUG
        CMAKE_C_FLAGS_RELEASE
        CMAKE_C_FLAGS_RELWITHDEBINFO
        CMAKE_C_FLAGS_MINSIZEREL
        )
  foreach(flags ${flags_configs})
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      if(NOT ${flags} MATCHES "-static-libstdc\\+\\+")
        set(${flags} "${${flags}} -static-libstdc++")
      endif()
      if(NOT ${flags} MATCHES "-static-libgcc")
        set(${flags} "${${flags}} -static-libgcc")
      endif()
    elseif(MSVC)
      if(${flags} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${flags} "${${flags}}")
      endif()
    endif()
  endforeach()
else()
  set(CAF_BUILD_STATIC_RUNTIME no)
endif()

# add helper target that simplifies re-running configure
if(NOT caf_is_subproject)
  add_custom_target(configure COMMAND ${CMAKE_CURRENT_BINARY_DIR}/config.status)
endif()

# Silence annoying MSVC warning.
if(MSVC)
  add_compile_options(/wd4275 /wd4251)
endif()

################################################################################
#                              utility functions                               #
################################################################################

# Appends `str` to the variable named `var` with a whitespace as separator.
# Suppresses a leading whitespace if the variable is empty and does nothing if
# `str` is empty.
function(build_string var str)
  if(NOT str STREQUAL "")
    if("${${var}}" STREQUAL "")
      set("${var}" "${str}" PARENT_SCOPE)
    else()
      set("${var}" "${${var}} ${str}" PARENT_SCOPE)
    endif()
  endif()
endfunction(build_string)

# Forces `var` to 'no' if the content of the variables evaluates to false.
function(pretty_no var)
  if(NOT "${${var}}")
    set("${var}" no PARENT_SCOPE)
  endif()
endfunction(pretty_no)

# Forces `var` to 'yes' if the content of the variables evaluates to false.
function(pretty_yes var)
  if("${${var}}")
    set("${var}" yes PARENT_SCOPE)
  endif()
endfunction(pretty_yes)

add_executable(caf-generate-enum-strings cmake/caf-generate-enum-strings.cpp)

function(enum_to_string relative_input_file relative_output_file)
  set(input "${CMAKE_CURRENT_SOURCE_DIR}/${relative_input_file}")
  set(output "${CMAKE_BINARY_DIR}/${relative_output_file}")
  add_custom_command(OUTPUT "${output}"
                     COMMAND caf-generate-enum-strings "${input}" "${output}"
                     DEPENDS caf-generate-enum-strings "${input}")
endfunction()

################################################################################
#                        set prefix paths if available                         #
################################################################################

build_string("CMAKE_PREFIX_PATH" "${CAF_QT_PREFIX_PATH}")

################################################################################
#                      enable ccache if required by user                       #
################################################################################

if(CAF_USE_CCACHE)
  find_program(CCACHE_PROGRAM ccache)
  if(CCACHE_PROGRAM)
    message(STATUS "Using ccache command: ${CCACHE_PROGRAM}")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
  else()
    message(STATUS "Unable to find ccache")
  endif()
endif(CAF_USE_CCACHE)

################################################################################
#      make sure config parameters are printed with yes or no in summary       #
################################################################################

pretty_yes("CAF_FORCE_NO_EXCEPTIONS")

pretty_no("CAF_ENABLE_RUNTIME_CHECKS")
pretty_no("CAF_NO_MEM_MANAGEMENT")
pretty_no("CAF_NO_EXCEPTIONS")
pretty_no("CAF_NO_OPENSSL")
pretty_no("CAF_NO_PYTHON")
pretty_no("CAF_NO_TOOLS")
pretty_no("CAF_NO_SUMMARY")

if(NOT CAF_NO_IO)
  set(CAF_NO_IO no)
else()
  set(CAF_NO_TOOLS yes)
  set(CAF_NO_PYTHON yes)
endif()

################################################################################
#                              get version of CAF                              #
################################################################################

# read content of config.hpp
file(READ "libcaf_core/caf/config.hpp" CONFIG_HPP)
# get line containing the version
string(REGEX MATCH "#define CAF_VERSION [0-9]+" VERSION_LINE "${CONFIG_HPP}")
# extract version number from line
string(REGEX MATCH "[0-9]+" VERSION_INT "${VERSION_LINE}")
# calculate major, minor, and patch version
math(EXPR CAF_VERSION_MAJOR "${VERSION_INT} / 10000")
math(EXPR CAF_VERSION_MINOR "( ${VERSION_INT} / 100) % 100")
math(EXPR CAF_VERSION_PATCH "${VERSION_INT} % 100")
# create full version string
set(CAF_VERSION
    "${CAF_VERSION_MAJOR}.${CAF_VERSION_MINOR}.${CAF_VERSION_PATCH}")
# set the library version for our shared library targets
if(CMAKE_HOST_SYSTEM_NAME MATCHES "OpenBSD")
  set(CAF_LIB_VERSION "${CAF_VERSION_MAJOR}.${CAF_VERSION_MINOR}")
else()
  set(CAF_LIB_VERSION "${CAF_VERSION}")
endif()

################################################################################
#   set output paths for binaries and libraries if not provided by the user    #
################################################################################

# prohibit in-source builds
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL "${CMAKE_CURRENT_BINARY_DIR}")
    message(FATAL_ERROR "In-source builds are not allowed. Please use "
                        "./configure to choose a build directory and "
                        "initialize the build configuration.")
endif()

if(NOT XCODE)
  # set binary output path if not defined by user
  if(NOT EXECUTABLE_OUTPUT_PATH)
    set(EXECUTABLE_OUTPUT_PATH "${CMAKE_CURRENT_BINARY_DIR}/bin")
  endif()
  # set library output path if not defined by user
  if(NOT LIBRARY_OUTPUT_PATH)
    set(LIBRARY_OUTPUT_PATH "${CMAKE_CURRENT_BINARY_DIR}/lib")
  endif()
endif()

################################################################################
#                                compiler setup                                #
################################################################################

# set optional build flags
# increase max. template depth on GCC and Clang
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang"
   OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  add_compile_options("-ftemplate-depth=512"
                      "-ftemplate-backtrace-limit=0")
endif()
# explicitly disable obnoxious GCC warnings
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  build_string("EXTRA_FLAGS" "-Wno-missing-field-initializers")
endif()
# add "-Werror" flag if --pedantic-build is used
if(CAF_CXX_WARNINGS_AS_ERRORS)
  build_string("EXTRA_FLAGS" "-Werror")
endif()
# set compiler flags for GCOV if requested
if(CAF_ENABLE_GCOV)
  if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(NO_INLINE "-fno-inline")
  else()
    set(NO_INLINE "-fno-inline -fno-inline-small-functions -fno-default-inline")
  endif()
  build_string("EXTRA_FLAGS" "-fprofile-arcs -ftest-coverage ${NO_INLINE}")
endif()
# set -fno-exception if requested
if(CAF_FORCE_NO_EXCEPTIONS)
  build_string("EXTRA_FLAGS" "-fno-exceptions")
endif()
# enable a ton of warnings if --more-clang-warnings is used
if(CAF_MORE_WARNINGS)
  if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(WFLAGS "-Weverything -Wno-c++98-compat -Wno-padded "
               "-Wno-documentation-unknown-command -Wno-exit-time-destructors "
               "-Wno-global-constructors -Wno-missing-prototypes "
               "-Wno-c++98-compat-pedantic -Wno-unused-member-function "
               "-Wno-unused-const-variable -Wno-switch-enum "
               "-Wno-abstract-vbase-init -Wno-shadow "
               "-Wno-missing-noreturn -Wno-covered-switch-default")
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(WFLAGS "-Waddress -Wall -Warray-bounds "
               "-Wattributes -Wbuiltin-macro-redefined -Wcast-align "
               "-Wcast-qual -Wchar-subscripts -Wclobbered -Wcomment "
               "-Wconversion -Wconversion-null -Wcoverage-mismatch "
               "-Wcpp -Wdelete-non-virtual-dtor -Wdeprecated "
               "-Wdeprecated-declarations -Wdiv-by-zero -Wdouble-promotion "
               "-Wempty-body -Wendif-labels -Wenum-compare -Wextra "
               "-Wfloat-equal -Wformat -Wfree-nonheap-object "
               "-Wignored-qualifiers -Winit-self "
               "-Winline -Wint-to-pointer-cast -Winvalid-memory-model "
               "-Winvalid-offsetof -Wlogical-op -Wmain -Wmaybe-uninitialized "
               "-Wmissing-braces -Wmultichar "
               "-Wnarrowing -Wnoexcept -Wnon-template-friend "
               "-Wnon-virtual-dtor -Wnonnull -Woverflow "
               "-Woverlength-strings -Wparentheses "
               "-Wpmf-conversions -Wpointer-arith -Wreorder "
               "-Wreturn-type -Wsequence-point "
               "-Wsign-compare -Wswitch -Wtype-limits -Wundef "
               "-Wuninitialized -Wunused -Wvla -Wwrite-strings")
  endif()
  # convert CMake list to a single string, erasing the ";" separators
  string(REPLACE ";" "" WFLAGS_STR ${WFLAGS})
  build_string("EXTRA_FLAGS" "${WFLAGS_STR}")
endif()
# allow enabling IPO on gcc/clang
if(POLICY CMP0069)
  cmake_policy(SET CMP0069 NEW)
else()
  if(CMAKE_INTERPROCEDURAL_OPTIMIZATION)
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      build_string("EXTRA_FLAGS" "-flto")
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
      build_string("EXTRA_FLAGS" "-flto -fno-fat-lto-objects")
    endif()
  endif()
endif()

# enable address sanitizer if requested by the user
if(CAF_SANITIZERS)
  add_compile_options("-fsanitize=${CAF_SANITIZERS}"
                      "-fno-omit-frame-pointer")
  list(APPEND CAF_EXTRA_LDFLAGS "-fsanitize=${CAF_SANITIZERS}")
endif()
# -pthread is ignored on MacOSX but required on other platforms
if(NOT APPLE AND NOT WIN32)
    build_string("EXTRA_FLAGS" "-pthread")
endif()
# -fPIC generates warnings on MinGW and Cygwin plus extra setup steps needed on MinGW
if(MINGW)
  add_definitions(-D_WIN32_WINNT=0x0600)
  add_definitions(-DWIN32)
  list(APPEND CAF_EXTRA_LDFLAGS -lws2_32 -liphlpapi -lpsapi)
  # build static to avoid runtime dependencies to GCC libraries
  build_string("EXTRA_FLAGS" "-static")
elseif(CYGWIN)
  build_string("EXTRA_FLAGS" "-U__STRICT_ANSI__")
endif()
if (WIN32)
  list(APPEND CAF_EXTRA_LDFLAGS ws2_32 iphlpapi)
endif()
# iOS support
if(CAF_OSX_SYSROOT)
  set(CMAKE_OSX_SYSROOT "${CAF_OSX_SYSROOT}")
endif()
if(CAF_IOS_DEPLOYMENT_TARGET)
  if(CAF_OSX_SYSROOT STREQUAL "iphonesimulator")
    build_string("EXTRA_FLAGS"
                 "-mios-simulator-version-min=${CAF_IOS_DEPLOYMENT_TARGET}")
  else()
    build_string("EXTRA_FLAGS"
                 "-miphoneos-version-min=${CAF_IOS_DEPLOYMENT_TARGET}")
  endif()
endif()
# check if the user provided CXXFLAGS, set defaults otherwise
if(NOT CMAKE_CXX_FLAGS)
  set(CMAKE_CXX_FLAGS                   "-Wextra -Wall -pedantic ${EXTRA_FLAGS}")
endif()
if(NOT CMAKE_CXX_FLAGS_DEBUG)
  set(CMAKE_CXX_FLAGS_DEBUG             "-O0 -g")
endif()
if(NOT CMAKE_CXX_FLAGS_MINSIZEREL)
  set(CMAKE_CXX_FLAGS_MINSIZEREL        "-Os")
endif()
if(NOT CMAKE_CXX_FLAGS_RELEASE)
  set(CMAKE_CXX_FLAGS_RELEASE           "-O3 -DNDEBUG")
endif()
if(NOT CMAKE_CXX_FLAGS_RELWITHDEBINFO)
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO    "-O2 -g -fno-omit-frame-pointer")
endif()
# set build default build type to RelWithDebInfo if not set
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif()
# needed by subprojects
if (DEFINED CMAKE_LD_LIBS)
  list(APPEND ${CMAKE_LD_LIBS})
endif()

################################################################################
#                                setup logging                                 #
################################################################################

# make sure log level is defined and all-uppercase
if(NOT CAF_LOG_LEVEL)
  set(CAF_LOG_LEVEL "QUIET")
else()
  string(TOUPPER "${CAF_LOG_LEVEL}" CAF_LOG_LEVEL)
endif()

set(validLogLevels QUIET ERROR WARNING INFO DEBUG TRACE)
list(FIND validLogLevels "${CAF_LOG_LEVEL}" logLevelIndex)
if(logLevelIndex LESS 0)
  MESSAGE(FATAL_ERROR "Invalid log level: \"${CAF_LOG_LEVEL}\"")
endif()

################################################################################
#                           setup for install target                           #
################################################################################

# install includes from test
install(DIRECTORY libcaf_test/caf/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/caf
        FILES_MATCHING PATTERN "*.hpp")

# process cmake_uninstall.cmake.in
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
               "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
               IMMEDIATE @ONLY)
# add uninstall target if it does not exist yet
if(NOT TARGET uninstall)
  add_custom_target(uninstall)
endif()
add_custom_command(TARGET uninstall
                   PRE_BUILD
                   COMMAND "${CMAKE_COMMAND}" -P
                   "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")

# -- set include paths for subprojects -----------------------------------------

configure_file("${PROJECT_SOURCE_DIR}/cmake/build_config.hpp.in"
               "${CMAKE_BINARY_DIR}/caf/detail/build_config.hpp"
               @ONLY)

# -- set include paths for all subprojects -------------------------------------

include_directories("${CMAKE_BINARY_DIR}"
                    "${CMAKE_CURRENT_SOURCE_DIR}/libcaf_core"
                    "${CMAKE_CURRENT_SOURCE_DIR}/libcaf_io"
                    "${CMAKE_CURRENT_SOURCE_DIR}/libcaf_test"
                    "${CMAKE_CURRENT_SOURCE_DIR}/libcaf_openssl")

################################################################################
#                                 add targets                                  #
################################################################################

macro(add_optional_caf_lib name)
  string(TOUPPER ${name} upper_name)
  set(flag_varname CAF_NO_${upper_name})
  if(NOT ${flag_varname}
     AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libcaf_${name}/CMakeLists.txt")
    add_subdirectory("libcaf_${name}")
  else()
    set(${flag_varname} yes)
  endif()
endmacro()

macro(add_optional_caf_binaries name)
  string(TOUPPER ${name} upper_name)
  set(dependency_failed no)
  # check all additional dependency flags
  foreach(flag_name ${ARGN})
    if(${flag_name})
      set(dependency_failed yes)
    endif()
  endforeach()
  if(NOT dependency_failed)
    if(NOT CAF_NO_${upper_name}
       AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${name}/CMakeLists.txt")
      add_subdirectory(${name})
    else()
      # make sure variable is set for nicer build log
      set(CAF_NO_${upper_name} yes)
    endif()
  else()
    message(STATUS
            "Disable ${name}, one of the following flags was set: ${ARGN}")
    # make sure variable is set for nicer build log
    set(CAF_NO_${upper_name} yes)
  endif()
endmacro()

# build core and I/O library
add_subdirectory(libcaf_core)
add_optional_caf_lib(io)

# build SSL module if OpenSSL library is available
if(NOT CAF_NO_OPENSSL)
  find_package(OpenSSL)
  if(OPENSSL_FOUND)
    # Check OpenSSL version >= 1.0.1
    if (OPENSSL_VERSION VERSION_LESS 1.0.1)
      message(STATUS
              "Disable OpenSSL. Required >= 1.0.1 due to TLSv1.2 support.")
      set(CAF_NO_OPENSSL yes)
    else()
      if(NOT CMAKE_CROSSCOMPILING)
        # Check if openssl headers and library versions match
        set(CMAKE_REQUIRED_LIBRARIES ${OPENSSL_LIBRARIES})
        set(CMAKE_REQUIRED_INCLUDES ${OPENSSL_INCLUDE_DIR})
        check_c_source_runs("
          #include <openssl/opensslv.h>
          #include <openssl/crypto.h>
          int main() {
            if (SSLeay() == OPENSSL_VERSION_NUMBER) {
              return 0;
            }
            return -1;
          }
        " OPENSSL_CORRECT_VERSION_NUMBER)
        if (NOT OPENSSL_CORRECT_VERSION_NUMBER)
          message(FATAL_ERROR
            "OpenSSL library version does not match headers")
        endif()
      endif()
      include_directories(BEFORE ${OPENSSL_INCLUDE_DIR})
      add_optional_caf_lib(openssl)
    endif()
  else(OPENSSL_FOUND)
    set(CAF_NO_OPENSSL yes)
  endif(OPENSSL_FOUND)
endif()

# build Python binding if not being told otherwise
if(NOT CAF_NO_PYTHON AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libcaf_python/CMakeLists.txt")
  add_subdirectory(libcaf_python)
endif()

# build examples if not being told otherwise
add_optional_caf_binaries(examples)

# build tools if not being told otherwise
add_optional_caf_binaries(tools)

export(EXPORT CAFTargets FILE CAFTargets.cmake NAMESPACE caf::)

install(EXPORT CAFTargets
        DESTINATION "${INSTALL_CAF_CMAKEDIR}"
        NAMESPACE caf::)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/CAFConfigVersion.cmake"
  VERSION ${CAF_VERSION}
  COMPATIBILITY ExactVersion)

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/CAFConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/CAFConfig.cmake" INSTALL_DESTINATION
  "${INSTALL_CAF_CMAKEDIR}")

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/CAFConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/CAFConfigVersion.cmake"
  DESTINATION "${INSTALL_CAF_CMAKEDIR}")

# -- Fetch branch name and SHA if available ------------------------------------

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/release.txt")
  file(READ "${CMAKE_CURRENT_SOURCE_DIR}/release.txt" CAF_RELEASE)
  string(REGEX REPLACE "\n" "" CAF_RELEASE "${CAF_RELEASE}")
else()
  set(CAF_RELEASE "${CAF_VERSION}")
endif()

message(STATUS "Set release version for all documentation to ${CAF_RELEASE}.")

# -- Setup for building manual and API documentation ---------------------------

if(NOT WIN32)
  add_subdirectory(doc)
endif()

################################################################################
#                     Add additional project files to GUI                      #
################################################################################

file(GLOB_RECURSE script_files "scripts/*")
add_custom_target(gui_dummy SOURCES configure ${script_files})

################################################################################
#                                print summary                                 #
################################################################################

# little helper macro to invert a boolean
macro(invertYesNo in out)
  if(${in})
    set(${out} no)
  else()
    set(${out} yes)
  endif()
endmacro()
# invert CAF_NO_* variables for nicer output
invertYesNo(CAF_NO_IO CAF_BUILD_IO)
invertYesNo(CAF_NO_EXAMPLES CAF_BUILD_EXAMPLES)
invertYesNo(CAF_NO_TOOLS CAF_BUILD_TOOLS)
invertYesNo(CAF_NO_UNIT_TESTS CAF_BUILD_UNIT_TESTS)
invertYesNo(CAF_NO_EXCEPTIONS CAF_BUILD_WITH_EXCEPTIONS)
invertYesNo(CAF_NO_MEM_MANAGEMENT CAF_BUILD_MEM_MANAGEMENT)
invertYesNo(CAF_NO_OPENSSL CAF_BUILD_OPENSSL)
invertYesNo(CAF_NO_PYTHON CAF_BUILD_PYTHON)
# collect all compiler flags
string(TOUPPER "${CMAKE_BUILD_TYPE}" UPPER_BUILD_TYPE)
set(ALL_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_${UPPER_BUILD_TYPE}}")
set(ALL_LD_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${CAF_EXTRA_LDFLAGS}")
string(STRIP "${ALL_LD_FLAGS}" ALL_LD_FLAGS)
# done
if(NOT CAF_NO_SUMMARY)
  message(STATUS
        "\n====================|  Build Summary  |===================="
        "\n"
        "\nCAF version:           ${CAF_VERSION}"
        "\n"
        "\nBuild type:            ${CMAKE_BUILD_TYPE}"
        "\nLink type:             ${LINK_TYPE}"
        "\nBuild static runtime:  ${CAF_BUILD_STATIC_RUNTIME}"
        "\nRuntime checks:        ${CAF_ENABLE_RUNTIME_CHECKS}"
        "\nLog level:             ${CAF_LOG_LEVEL}"
        "\nWith mem. mgmt.:       ${CAF_BUILD_MEM_MANAGEMENT}"
        "\nWith exceptions:       ${CAF_BUILD_WITH_EXCEPTIONS}"
        "\n"
        "\nBuild I/O module:      ${CAF_BUILD_IO}"
        "\nBuild tools:           ${CAF_BUILD_TOOLS}"
        "\nBuild examples:        ${CAF_BUILD_EXAMPLES}"
        "\nBuild unit tests:      ${CAF_BUILD_UNIT_TESTS}"
        "\nBuild OpenSSL:         ${CAF_BUILD_OPENSSL}"
        "\nBuild Python:          ${CAF_BUILD_PYTHON}"
        "\n"
        "\nCXX:                   ${CMAKE_CXX_COMPILER}"
        "\nCXXFLAGS:              ${ALL_CXX_FLAGS}"
        "\nLINKER_FLAGS (shared)  ${ALL_LD_FLAGS}"
        "\n"
        "\nSource directory:      ${CMAKE_CURRENT_SOURCE_DIR}"
        "\nBuild directory:       ${CMAKE_CURRENT_BINARY_DIR}"
        "\nExecutable path:       ${EXECUTABLE_OUTPUT_PATH}"
        "\nLibrary path:          ${LIBRARY_OUTPUT_PATH}"
        "\nInstall prefix:        ${CMAKE_INSTALL_PREFIX}"
        "\nGenerator:             ${CMAKE_GENERATOR}"
        "\n"
        "\n===========================================================\n")
endif()
