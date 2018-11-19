cmake_minimum_required(VERSION 3.2)

include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.23.142.tar.gz"
    SHA1 "1d841ca74150c92054896a7f6d88a123dd9e615d"
)

project(dmlc VERSION 0.3)

if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/build/private/local_config.cmake)
  include(${CMAKE_CURRENT_SOURCE_DIR}/build/private/local_config.cmake)
endif()

set(CMAKE_LOCAL "${PROJECT_SOURCE_DIR}/cmake")
list(APPEND CMAKE_MODULE_PATH ${CMAKE_LOCAL}/Modules)

include(CheckCXXSymbolExists)
include(cmake/Utils.cmake)
#include(cmake/dmlccore.cmake)

# Options
dmlccore_option(USE_HDFS "Build with HDFS support" OFF)
dmlccore_option(USE_AZURE "Build with AZURE support" OFF)
dmlccore_option(USE_S3 "Build with S3 support" OFF)
dmlccore_option(USE_OPENMP "Build with OpenMP" ON)
dmlccore_option(USE_CXX14_IF_AVAILABLE "Build with C++14 if the compiler supports it" OFF)
dmlccore_option(GOOGLE_TEST "Build google tests" OFF)

# include path
set(INCLUDE_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(INCLUDE_DMLC_DIR "${INCLUDE_ROOT}/dmlc")
include_directories("${INCLUDE_ROOT}")

set(dmlccore_LINKER_LIBS "")
# HDFS configurations
if(USE_HDFS)
 find_package(JNI REQUIRED)
 list(APPEND dmlccore_LINKER_LIBS ${JAVA_JVM_LIBRARY})

 hunter_add_package(hdf5)
 find_package(hdf5 CONFIG REQUIRED)
 list(APPEND dmlccore_LINKER_LIBS hdf5)
 add_definitions(-DDMLC_USE_HDFS=1)
else()
 add_definitions(-DDMLC_USE_HDFS=0)
endif()
# S3 configurations
if(USE_S3)
 hunter_add_package(CURL)
 find_package(CURL CONFIG REQUIRED)
 list(APPEND dmlccore_LINKER_LIBS CURL::libcurl)

 hunter_add_package(OpenSSL REQUIRED)
 find_package(OpenSSL REQUIRED)
 list(APPEND dmlccore_LINKER_LIBS OpenSSL::SSL OpenSSL::Crypto)

 add_definitions(-DDMLC_USE_S3=1)
else()
 add_definitions(-DDMLC_USE_S3=0)
endif()
# Azure configurations
if(USE_AZURE)
  add_definitions(-DDMLC_USE_AZURE=1)
else()
  add_definitions(-DDMLC_USE_AZURE=0)
endif()

if(WIN32 AND (NOT MSVC))  # On Windows, link Shlwapi.lib for non-MSVC compilers
  list(APPEND dmlccore_LINKER_LIBS Shlwapi)
endif()

# Check location of clock_gettime; if it's in librt, link it
include(CheckLibraryExists)
CHECK_LIBRARY_EXISTS(rt clock_gettime "time.h" HAVE_CLOCK_GETTIME_IN_LIBRT)
if(HAVE_CLOCK_GETTIME_IN_LIBRT)
  list(APPEND dmlccore_LINKER_LIBS rt)
endif()

# Older stdc++ enable c++11 items
add_definitions(-D__USE_XOPEN2K8)

# Check headers and symbols
include(CheckSymbolExists)
include(CheckIncludeFile)
include(CheckIncludeFileCXX)
check_cxx_symbol_exists(fopen64 stdio.h DMLC_FOPEN_64_PRESENT)
check_include_file_cxx(cxxabi.h DMLC_CXXABI_H_PRESENT)
check_symbol_exists(nanosleep time.h DMLC_NANOSLEEP_PRESENT)

# Check existence of backtrace(3)
find_package(Backtrace)
if(Backtrace_FOUND)
  set(DMLC_EXECINFO_H_PRESENT 1)
  set(DMLC_EXECINFO_H ${Backtrace_HEADER})
  include_directories(${Backtrace_INCLUDE_DIRS})
  list(APPEND dmlccore_LINKER_LIBS ${Backtrace_LIBRARIES})
else()
  set(DMLC_EXECINFO_H_PRESENT 0)
endif()

# Check endianness
include(TestBigEndian)
test_big_endian(BIG_ENDIAN)
if(BIG_ENDIAN)
  set(DMLC_CMAKE_LITTLE_ENDIAN 0)
else()
  set(DMLC_CMAKE_LITTLE_ENDIAN 1)
endif()

message(STATUS "${CMAKE_LOCAL}/build_config.h.in -> ${INCLUDE_DMLC_DIR}/build_config.h")
configure_file("${CMAKE_LOCAL}/build_config.h.in" "${INCLUDE_DMLC_DIR}/build_config.h")

# compiler flags
if(MSVC)
  add_definitions(-DDMLC_USE_CXX11)
else(MSVC)
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)
  include(CheckCXXCompilerFlag)
  if(USE_CXX14_IF_AVAILABLE)
    check_cxx_compiler_flag("-std=c++14"    SUPPORT_CXX14)
  endif()
  check_cxx_compiler_flag("-std=c++11"    SUPPORT_CXX11)
  check_cxx_compiler_flag("-std=c++0x"    SUPPORT_CXX0X)
  check_cxx_compiler_flag("-msse2"        SUPPORT_MSSE2)
  set(CMAKE_C_FLAGS "-Wall -Wno-unknown-pragmas -fPIC")
  if(CMAKE_BUILD_TYPE STREQUAL "DEBUG")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0")
  elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0")
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3")
  endif()
  if(SUPPORT_MSSE2)
  	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse2")
  endif()
  set(CMAKE_CXX_FLAGS ${CMAKE_C_FLAGS})
  if(SUPPORT_CXX14)
    add_definitions(-DDMLC_USE_CXX14=1)
    add_definitions(-DDMLC_USE_CXX11=1)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
  elseif(SUPPORT_CXX11)
    add_definitions(-DDMLC_USE_CXX11=1)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  elseif(SUPPORT_CXX0X)
    add_definitions(-DDMLC_USE_CXX11=1)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
  endif()
endif(MSVC)

if(USE_OPENMP)
  if(MSVC)
    set(OPENMP_FOUND ON)
    set(OpenMP_C_FLAGS "-openmp")
    set(OpenMP_CXX_FLAGS ${OpenMP_C_FLAGS})
  else()
    find_package(OpenMP REQUIRED)
  endif()
  if(OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
  endif()
endif()

FILE(GLOB SOURCE "src/*.cc")
FILE(GLOB_RECURSE SOURCE_INCLUDE "include/*")
list(APPEND SOURCE ${SOURCE_INCLUDE})
list(APPEND SOURCE "src/io/line_split.cc")
list(APPEND SOURCE "src/io/recordio_split.cc")
list(APPEND SOURCE "src/io/indexed_recordio_split.cc")
list(APPEND SOURCE "src/io/input_split_base.cc")
list(APPEND SOURCE "src/io/filesys.cc")
list(APPEND SOURCE "src/io/local_filesys.cc")

if(USE_HDFS)
  list(APPEND SOURCE "src/io/hdfs_filesys.cc")
endif()
if(USE_S3)
  list(APPEND SOURCE "src/io/s3_filesys.cc")
endif()
if(USE_AZURE)
  list(APPEND SOURCE "src/io/azure_filesys.cc")
endif()

add_library(dmlc ${SOURCE})
target_link_libraries(dmlc ${dmlccore_LINKER_LIBS})
target_compile_definitions(dmlc PRIVATE -D_XOPEN_SOURCE=700
  -D_POSIX_SOURCE -D_POSIX_C_SOURCE=200809L -D_DARWIN_C_SOURCE)

# ---[ Linter target
if(MSVC)
  find_package(PythonInterp)
  set(PYTHON_EXECUTABLE ${PYTHON_EXECUTABLE} CACHE FILEPATH "Path to the python 2.x executable")
endif()
set(LINT_DIRS include src scripts)
add_custom_target(dmlc_lint COMMAND ${CMAKE_COMMAND} -DMSVC=${MSVC} -DPYTHON_EXECUTABLE=${PYTHON_EXECUTABLE}  -DPROJECT_SOURCE_DIR=${PROJECT_SOURCE_DIR} -DLINT_DIRS=${LINT_DIRS} -DPROJECT_NAME=dmlc -P ${PROJECT_SOURCE_DIR}/cmake/lint.cmake)

# Setup testing
if(GOOGLE_TEST)
  include(CTest)
  add_subdirectory(test/unittest)
endif()

# Installation (https://github.com/forexample/package-example) {

# Layout. This works for all platforms:
#   * <prefix>/lib/cmake/<PROJECT-NAME>
#   * <prefix>/lib/
#   * <prefix>/include/
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# Include module with fuction 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Configure '<PROJECT-NAME>ConfigVersion.cmake'
# Use:
#   * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Configure '<PROJECT-NAME>Config.cmake'
# Use variables:
#   * TARGETS_EXPORT_NAME
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

# Targets:
#   * <prefix>/lib/libdmlc.a
#   * header location after install: <prefix>/include/dmlc/dmlc.h
#   * headers can be included by C++ code `#include <dmlc/dmlc.h>`
install(
    TARGETS dmlc
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

# Headers:
install(
    DIRECTORY "include/"
    DESTINATION "${include_install_dir}"
    FILES_MATCHING PATTERN "*.h"
)

# Config
#   * <prefix>/lib/cmake/dmlc/dmlcConfig.cmake
#   * <prefix>/lib/cmake/dmlc/dmlcConfigVersion.cmake
install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

# Config
#   * <prefix>/lib/cmake/Foo/FooTargets.cmake
install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
# }
