cmake_minimum_required(VERSION 3.24)
include(FetchContent)

# pre-cache the linker flags
set(CMAKE_EXE_LINKER_FLAGS_INIT "-sNODERAWFS=1 -sFORCE_FILESYSTEM=1 -sSINGLE_FILE=1 --extern-pre-js=${CMAKE_SOURCE_DIR}/shebang.txt" CACHE INTERNAL "")
set(HDF5_VERSION "1_12_3" CACHE STRING "HDF5 version to build")
set(CMAKE_POSITION_INDEPENDENT_CODE ON CACHE BOOL "use -fPIC for all libs" FORCE)
set(CMAKE_EXECUTABLE_SUFFIX_C "" CACHE STRING "" FORCE)

set(SRC_HASH_1_12_3 39e2e3e25f2263f2dc3370556c174952675d7ac51c4d6f7d4f5e8b3a8bd10ac6)
set(SRC_HASH_1.14.5 ec2e13c52e60f9a01491bb3158cb3778c985697131fc6a342262d32a26e58e44)
set(SRC_HASH_1.14.6 6df53f84376d87e96eeb255703e3b1a348fc08a48e06e37053f1bfb379970189)
set(SRC_HASH_1_10_11 4ef6375fc7d8c54dcd66e9bc35a7a3580d33cd8878bdf21ad1eb388a43863159)

if( NOT DEFINED SRC_HASH_${HDF5_VERSION})
  message( FATAL_ERROR "Version ${HDF5_VERSION} is not supported.  See CMakeLists.txt for valid versions" )
endif()

message(STATUS "Building version ${HDF5_VERSION}...")

# Collect all patch files
file(GLOB PATCH_FILES LIST_DIRECTORIES false
    "${CMAKE_CURRENT_SOURCE_DIR}/patches/${HDF5_VERSION}/*.patch"
)

message(STATUS "Patches found: ${PATCH_FILES}")
# Create patch command if patches exist
set (patch_files_command)
if(PATCH_FILES)
    foreach(patch_file ${PATCH_FILES})
        list(APPEND patch_files_command
           COMMAND git apply --ignore-whitespace "${patch_file}"
        )
    endforeach()
endif()

FetchContent_Declare(
  hdf5
  GIT_REPOSITORY https://github.com/HDFGroup/hdf5.git
  GIT_TAG hdf5-${HDF5_VERSION}
  PATCH_COMMAND ${patch_files_command}
  UPDATE_DISCONNECTED 1
)
FetchContent_GetProperties(hdf5)
if (NOT hdf5_POPULATED)
  FetchContent_Populate(hdf5)
endif()

FetchContent_Declare(
  zlib
  URL http://zlib.net/fossils/zlib-1.3.tar.gz
  URL_HASH  SHA256=ff0ba4c292013dbc27530b3a81e1f9a813cd39de01ca5e0f8bf355702efa593e
  OVERRIDE_FIND_PACKAGE
)
FetchContent_Populate(zlib)

# add global included path because zlib not set up to be subproject
include_directories(${zlib_SOURCE_DIR} ${zlib_BINARY_DIR})
add_subdirectory(${zlib_SOURCE_DIR} ${zlib_BINARY_DIR})
# FetchContent_MakeAvailable(zlib)
set(ZLIB_INCLUDE_DIR ${zlib_SOURCE_DIR})
set(ZLIB_LIBRARY ${zlib_BINARY_DIR}/libz.a)

FetchContent_Declare(
  libaec
  GIT_REPOSITORY https://gitlab.dkrz.de/k202009/libaec
  GIT_TAG v1.0.6
)
set(libaec_USE_STATIC_LIBS ON)
FetchContent_MakeAvailable(libaec)

set (USE_LIBAEC ON CACHE BOOL "Use libaec szip replacement" FORCE)
set (USE_LIBAEC_STATIC ON CACHE BOOL "Use libaec szip replacement" FORCE)
set (SZIP_INCLUDE_DIR ${libaec_SOURCE_DIR}/include)
set (SZIP_LIBRARY ${libaec_BINARY_DIR}/src/libsz.a)
set (libaec_DIR ${libaec_BINARY_DIR}/cmake)
set (libaec_ROOT ${libaec_BINARY_DIR})
# find_package(libaec 1.0.5 REQUIRED CONFIG)
set (libaec_INCLUDE_DIR ${libaec_SOURCE_DIR}/include)
set (libaec_LIBRARY ${libaec_BINARY_DIR}/libaec.a)

# set the project name
project(libhdf5-wasm-build)

## This doesn't work because of naming of libszaec vs. libsz
## The USE_LIBAEC=ON setting sets the wrong library name (libszaec),
## while USE_LIBAEC=OFF sets the wrong include_dir (should be SOURCE/include instead of SOURCE/src)
# set(HDF5_ALLOW_EXTERNAL_SUPPORT "GIT" CACHE STRING "" FORCE)
# set(ZLIB_GIT_URL https://github.com/madler/zlib)
# set(ZLIB_GIT_BRANCH v1.3)
# set(SZIP_GIT_URL https://gitlab.dkrz.de/k202009/libaec)
# set(SZIP_GIT_BRANCH v1.0.6)

set (BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
option(BUILD_SHARED_LIBS "Build shared libs" OFF)
option(HDF5_BUILD_EXAMPLES "Build Examples" OFF)
option(HDF5_BUILD_TOOLS "Build Tools" OFF)
option(HDF5_BUILD_UTILS "Build Utils" OFF)
option(HDF5_BUILD_CPP_LIB "Build CPP libraries" ON)
option(HDF5_ENABLE_Z_LIB_SUPPORT "Enable ZLIB" ON)
option(HDF5_ENABLE_SZIP_SUPPORT "Enable SZIP" ON)
option(SZIP_USE_EXTERNAL "needs to be off" 0)
option(ZLIB_USE_EXTERNAL "needs to be off" 0)

set (BUILD_TESTING OFF CACHE BOOL "Do not build tests by default" FORCE)
set(HDF5_EXTERNALLY_CONFIGURED 1)
set(H5_HAVE_GETPWUID OFF)
set(H5_HAVE_SIGNAL OFF)

set(CMAKE_BUILD_TYPE RELEASE)

add_subdirectory(${hdf5_SOURCE_DIR} ${hdf5_BINARY_DIR})
install(EXPORT hdf5-targets DESTINATION cmake)
install (
    FILES ${hdf5_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/hdf5-config.cmake
    DESTINATION cmake
    COMPONENT configinstall
)
