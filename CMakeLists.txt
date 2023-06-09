# vim: et ts=4 sts=4 sw=4 tw=0

set(JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION "3.13.2")
cmake_minimum_required(VERSION ${JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION})
cmake_policy(VERSION ${JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION})
if(POLICY CMP0091)
    cmake_policy(SET CMP0091 NEW)
endif()

# Build the library with C++11 standard support, independent from other including
# software which may use a different CXX_STANDARD or CMAKE_CXX_STANDARD.
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Ensure that CMAKE_BUILD_TYPE has a value specified for single configuration generators.
if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING
        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel Coverage.")
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# ---------------------------------------------------------------------------
# use ccache if found, has to be done before project()
# ---------------------------------------------------------------------------
find_program(CCACHE_EXECUTABLE "ccache" HINTS /usr/local/bin /opt/local/bin)
if(CCACHE_EXECUTABLE)
    message(STATUS "use ccache")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_EXECUTABLE}" CACHE PATH "ccache" FORCE)
    set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_EXECUTABLE}" CACHE PATH "ccache" FORCE)
endif()

project(jsoncpp
        # Note: version must be updated in three places when doing a release. This
        # annoying process ensures that CMake and the headers report the correct version.
        # 1. ./include/json/version.h
        # 2. ./CMakeLists.txt
        # IMPORTANT: also update the PROJECT_SOVERSION!!
        VERSION 1.9.5 # <major>[.<minor>[.<patch>[.<tweak>]]]
        LANGUAGES CXX)

message(STATUS "JsonCpp Version: ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")
set(PROJECT_SOVERSION 25)

include(${CMAKE_CURRENT_SOURCE_DIR}/include/PreventInSourceBuilds.cmake)
include(${CMAKE_CURRENT_SOURCE_DIR}/include/PreventInBuildInstalls.cmake)

option(JSONCPP_WITH_TESTS "Compile and (for jsoncpp_check) run JsonCpp test executables" ON)
option(JSONCPP_WITH_POST_BUILD_UNITTEST "Automatically run unit-tests as a post build step" ON)
option(JSONCPP_WITH_WARNING_AS_ERROR "Force compilation to fail if a warning occurs" OFF)
option(JSONCPP_WITH_STRICT_ISO "Issue all the warnings demanded by strict ISO C and ISO C++" ON)
option(JSONCPP_WITH_EXAMPLE "Compile JsonCpp example" OFF)
option(JSONCPP_STATIC_WINDOWS_RUNTIME "Use static (MT/MTd) Windows runtime" OFF)
option(BUILD_SHARED_LIBS "Build jsoncpp_lib as a shared library." ON)
option(BUILD_STATIC_LIBS "Build jsoncpp_lib as a static library." ON)
option(BUILD_OBJECT_LIBS "Build jsoncpp_lib as a object library." ON)

# Adhere to GNU filesystem layout conventions
include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib" CACHE PATH "Archive output dir.")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib" CACHE PATH "Library output dir.")
set(CMAKE_PDB_OUTPUT_DIRECTORY     "${CMAKE_BINARY_DIR}/bin" CACHE PATH "PDB (MSVC debug symbol)output dir.")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin" CACHE PATH "Executable/dll output dir.")

macro(use_compilation_warning_as_error)
    if(MSVC)
        # Only enabled in debug because some old versions of VS STL generate
        # warnings when compiled in release configuration.
        add_compile_options($<$<CONFIG:Debug>:/WX>)
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        add_compile_options(-Werror)
        if(JSONCPP_WITH_STRICT_ISO)
            add_compile_options(-pedantic-errors)
        endif()
    endif()
endmacro()

# Include our configuration header
include_directories(${jsoncpp_SOURCE_DIR}/include)

if(MSVC)
    # Only enabled in debug because some old versions of VS STL generate
    # unreachable code warning when compiled in release configuration.
    add_compile_options($<$<CONFIG:Debug>:/W4>)
    if (JSONCPP_STATIC_WINDOWS_RUNTIME)
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    # using regular Clang or AppleClang
    add_compile_options(-Wall -Wconversion -Wshadow)

    if(JSONCPP_WITH_WARNING_AS_ERROR)
        add_compile_options(-Werror=conversion -Werror=sign-compare)
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # using GCC
    add_compile_options(-Wall -Wconversion -Wshadow -Wextra)
    # not yet ready for -Wsign-conversion

    if(JSONCPP_WITH_STRICT_ISO)
        add_compile_options(-Wpedantic)
    endif()
    if(JSONCPP_WITH_WARNING_AS_ERROR)
        add_compile_options(-Werror=conversion)
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
    # using Intel compiler
    add_compile_options(-Wall -Wconversion -Wshadow -Wextra)

    if(JSONCPP_WITH_WARNING_AS_ERROR)
        add_compile_options(-Werror=conversion)
    elseif(JSONCPP_WITH_STRICT_ISO)
        add_compile_options(-Wpedantic)
    endif()
endif()

if(JSONCPP_WITH_WARNING_AS_ERROR)
    use_compilation_warning_as_error()
endif()

if(JSONCPP_WITH_TESTS)
    enable_testing()
    include(CTest)
endif()

# Build the different applications
add_subdirectory(src)

#install the example
if(JSONCPP_WITH_EXAMPLE)
    add_subdirectory(example)
endif()
