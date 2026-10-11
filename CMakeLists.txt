cmake_minimum_required(VERSION 3.28)

# The current implementation is compiled only for sm_120a. Set and validate the
# architecture before CUDA compiler detection so CMake never injects a generic
# host-toolchain default or accepts an unsupported target.
if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
  set(CMAKE_CUDA_ARCHITECTURES 120a CACHE STRING "CUDA architectures to build")
endif()
if(NOT CMAKE_CUDA_ARCHITECTURES STREQUAL "120a")
  message(FATAL_ERROR
    "NInfer supports only CMAKE_CUDA_ARCHITECTURES=120a; got "
    "'${CMAKE_CUDA_ARCHITECTURES}'")
endif()

project(ninfer LANGUAGES C CXX CUDA)

# --- Global settings ---------------------------------------------------------
option(NINFER_BUILD_APPS "Build the NInfer CLI and server" ON)
option(BUILD_TESTING "Build NInfer tests" OFF)
option(NINFER_BUILD_BENCHMARKS "Build NInfer benchmarks" OFF)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_STANDARD 20)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)        # clangd / IDE tooling
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

if(CMAKE_GENERATOR MATCHES "Ninja")
  set_property(GLOBAL PROPERTY JOB_POOLS cuda_link=1)
  set(CMAKE_JOB_POOL_LINK cuda_link)
endif()

if(CMAKE_CUDA_COMPILER_VERSION VERSION_LESS 13.1)
  message(FATAL_ERROR
    "NInfer requires CUDA 13.1 or newer; found CUDA compiler "
    "${CMAKE_CUDA_COMPILER_VERSION}")
endif()

set(NINFER_BUILD_MEDIA_ACQUIRE OFF)
if(NINFER_BUILD_APPS OR BUILD_TESTING)
  set(NINFER_BUILD_MEDIA_ACQUIRE ON)
endif()

set(NINFER_BUILD_PROMPT_INPUT OFF)
if(NINFER_BUILD_APPS)
  set(NINFER_BUILD_PROMPT_INPUT ON)
endif()

set(NINFER_BUILD_SERVE OFF)
if(NINFER_BUILD_APPS OR BUILD_TESTING)
  set(NINFER_BUILD_SERVE ON)
endif()

find_package(CUDAToolkit REQUIRED)
# --- FFMPEG (direct lookup, no pkg-config needed) ---------------------------
find_path(FFMPEG_INCLUDE_DIR
  NAMES libavformat/avformat.h
  PATHS C:/ffmpeg/include C:/msys64/mingw64/include $ENV{FFMPEG_DIR}/include)
find_library(AVFORMAT_LIB  NAMES avformat avformat-60  PATHS C:/ffmpeg/lib C:/msys64/mingw64/lib $ENV{FFMPEG_DIR}/lib)
find_library(AVCODEC_LIB   NAMES avcodec   avcodec-60   PATHS C:/ffmpeg/lib C:/msys64/mingw64/lib $ENV{FFMPEG_DIR}/lib)
find_library(AVUTIL_LIB    NAMES avutil    avutil-58    PATHS C:/ffmpeg/lib C:/msys64/mingw64/lib $ENV{FFMPEG_DIR}/lib)
find_library(SWSCALE_LIB   NAMES swscale   swscale-7    PATHS C:/ffmpeg/lib C:/msys64/mingw64/lib $ENV{FFMPEG_DIR}/lib)

if(NOT FFMPEG_INCLUDE_DIR OR NOT AVFORMAT_LIB OR NOT AVCODEC_LIB OR NOT AVUTIL_LIB OR NOT SWSCALE_LIB)
  message(FATAL_ERROR "FFMPEG not found. Set FFMPEG_DIR or install FFMPEG.")
endif()
add_library(PkgConfig::FFMPEG INTERFACE IMPORTED)
set_target_properties(PkgConfig::FFMPEG PROPERTIES
  INTERFACE_INCLUDE_DIRECTORIES "${FFMPEG_INCLUDE_DIR}"
  INTERFACE_LINK_LIBRARIES     "${AVFORMAT_LIB};${AVCODEC_LIB};${AVUTIL_LIB};${SWSCALE_LIB}")

# --- libCurl ------------------------------------------------------------------
if(NINFER_BUILD_MEDIA_ACQUIRE)
  find_package(CURL MODULE REQUIRED)
  add_library(PkgConfig::LIBCURL ALIAS CURL::libcurl)
endif()
find_package(Threads REQUIRED)

# --- Subdirectories ----------------------------------------------------------
add_subdirectory(src)
if(NINFER_BUILD_APPS)
  add_subdirectory(apps)
endif()
if(BUILD_TESTING)
  enable_testing()
  add_subdirectory(tests)
endif()
if(NINFER_BUILD_BENCHMARKS)
  add_subdirectory(bench)
endif()


