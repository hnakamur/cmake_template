cmake_minimum_required(VERSION 3.21)

# This template attempts to be "fetch_content"-able
# so that it works well with tools like CPM or other
# manual dependency management

# Only set the cxx_standard if it is not set by someone else
if(NOT DEFINED CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 23)
endif()

# strongly encouraged to enable this globally to avoid conflicts between
# -Wpedantic being enabled and -std=c++20 and -std=gnu++20 for example
# when compiling with PCH enabled
set(CMAKE_CXX_EXTENSIONS OFF)

# Set the project name and language
project(
  myproject
  VERSION 0.0.1
  DESCRIPTION ""
  HOMEPAGE_URL "%%myurl%%"
  LANGUAGES CXX C
)

include(cmake/PreventInSourceBuilds.cmake)
include(ProjectOptions.cmake)

myproject_setup_options()

myproject_global_options()
include(Dependencies.cmake)
myproject_setup_dependencies()

myproject_local_options()

# don't know if this should be set globally from here or not...
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

set(GIT_SHA
    "Unknown"
    CACHE STRING "SHA this build was generated from"
)
string(SUBSTRING "${GIT_SHA}" 0 8 GIT_SHORT_SHA)

target_compile_features(myproject_options INTERFACE cxx_std_${CMAKE_CXX_STANDARD})

add_library(myproject::myproject_options ALIAS myproject_options)
add_library(myproject::myproject_warnings ALIAS myproject_warnings)

#add_library(myproject::myproject_options INTERFACE IMPORTED)
#add_library(myproject::myproject_warnings INTERFACE IMPORTED)

# configure files based on CMake configuration options
add_subdirectory(configured_files)

# Adding the src:
add_subdirectory(src)

# Don't even look at tests if we're not top level
if(NOT PROJECT_IS_TOP_LEVEL)
  return()
endif()

# Adding the tests:
include(CTest)

if(BUILD_TESTING)
  message(AUTHOR_WARNING "Building Tests. Be sure to check out test/constexpr_tests.cpp for constexpr testing")
  add_subdirectory(test)
endif()

if(myproject_BUILD_FUZZ_TESTS)
  message(AUTHOR_WARNING "Building Fuzz Tests, using fuzzing sanitizer https://www.llvm.org/docs/LibFuzzer.html")
  if(NOT myproject_ENABLE_SANITIZER_ADDRESS AND NOT myproject_ENABLE_SANITIZER_THREAD)
    message(WARNING "You need asan or tsan enabled for meaningful fuzz testing")
  endif()
  add_subdirectory(fuzz_test)

endif()

# If MSVC is being used, and ASAN is enabled, we need to set the debugger environment
# so that it behaves well with MSVC's debugger, and we can run the target from visual studio
if(MSVC)
  get_all_installable_targets(all_targets)
  message("all_targets=${all_targets}")
  set_target_properties(${all_targets} PROPERTIES VS_DEBUGGER_ENVIRONMENT "PATH=$(VC_ExecutablePath_x64);%PATH%")
endif()

# set the startup project for the "play" button in MSVC
set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT intro)

if(CMAKE_SKIP_INSTALL_RULES)
  return()
endif()

include(cmake/clang_format.cmake)
add_clang_format_target(clang_format_src src)
add_clang_format_target(clang_format_tests test)
add_custom_target(clang_format DEPENDS clang_format_src clang_format_tests)

include(cmake/cmake_format.cmake)
add_cmake_format_target_toplevel(cmake_format_toplevel)
add_cmake_format_target(cmake_format_cmake_dir cmake)
add_cmake_format_target(cmake_format_src_dir src)
add_cmake_format_target(cmake_format_tests_dir test)
add_custom_target(
  cmake_format DEPENDS cmake_format_toplevel cmake_format_cmake_dir cmake_format_src_dir cmake_format_tests_dir
)

# NOTE: Remove ALL below if you do not want to execute the formatting target every time.
add_custom_target(format ALL DEPENDS clang_format cmake_format)
