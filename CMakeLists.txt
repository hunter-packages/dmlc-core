cmake_minimum_required(VERSION 3.1) # for Threads::Threads

include("cmake/HunterGate.cmake")
HunterGate(
  URL "https://github.com/ruslo/hunter/archive/v0.19.210.tar.gz"
  SHA1 "87eb902de6ca89e1dafc71e475344f523ef67729"
)

project(dmlc VERSION 0.0.0)

if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/build/private/local_config.cmake)
  include(${CMAKE_CURRENT_SOURCE_DIR}/build/private/local_config.cmake)
endif()

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)

include(cmake/Utils.cmake)
#include(cmake/dmlccore.cmake)

# Options
dmlccore_option(USE_HDFS "Build with HDFS support" OFF)
dmlccore_option(USE_AZURE "Build with AZURE support" OFF)
dmlccore_option(USE_S3 "Build with S3 support" OFF)
dmlccore_option(USE_OPENMP "Build with OpenMP" ON)

dmlccore_option(BUILD_EXAMPLES "Build examples" OFF)
dmlccore_option(BUILD_TESTS "Build examples" OFF)
dmlccore_option(BUILD_UNIT_TESTS "Build examples" OFF)

# include path
include_directories("include")

set(dmlccore_LINKER_LIBS "")
# HDFS configurations
if(USE_HDFS)

 find_package(JNI REQUIRED)
 list(APPEND dmlccore_LINKER_LIBS ${JAVA_JVM_LIBRARY})
 
 hunter_add_package(hdf5 CONFIG REQUIRED) # use hdf5
 find_package(hdf5 REQUIRED)
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

# Older stdc++ enable c++11 items
add_definitions(-D__USE_XOPEN2K8)

# compile
if(MSVC)
  add_definitions(-DDMLC_USE_CXX11)
else(MSVC)
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)
  include(CheckCXXCompilerFlag)
  check_cxx_compiler_flag("-std=c++11"   SUPPORT_CXX11)
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
  if(SUPPORT_CXX11)
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

if(BUILD_EXAMPLES)
  add_subdirectory(example)
endif()

if(BUILD_TESTS)
  add_subdirectory(test)
endif()

if(BUILD_UNIT_TESTS)
  enable_testing()
  add_subdirectory(test/unittest)
endif()

# ---[ Linter target
if(MSVC)
  find_package(PythonInterp)
  set(PYTHON_EXECUTABLE ${PYTHON_EXECUTABLE} CACHE FILEPATH "Path to the python 2.x executable")
endif()
set(LINT_DIRS include src scripts)
add_custom_target(dmlc_lint COMMAND ${CMAKE_COMMAND} -DMSVC=${MSVC} -DPYTHON_EXECUTABLE=${PYTHON_EXECUTABLE}  -DPROJECT_SOURCE_DIR=${PROJECT_SOURCE_DIR} -DLINT_DIRS=${LINT_DIRS} -DPROJECT_NAME=dmlc -P ${PROJECT_SOURCE_DIR}/cmake/lint.cmake)

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
