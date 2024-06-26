# ##############################################################################
# Copyright (C) Intel Corporation
#
# SPDX-License-Identifier: MIT
# ##############################################################################
cmake_minimum_required(VERSION 3.13.0)
project(vpl-infer)

# Default install places 64 bit runtimes in the environment, so we want to do a
# 64 bit build by default.
if(WIN32)
  if(NOT DEFINED CMAKE_GENERATOR_PLATFORM)
    set(CMAKE_GENERATOR_PLATFORM
        x64
        CACHE STRING "")
    message(STATUS "Generator Platform set to ${CMAKE_GENERATOR_PLATFORM}")
  endif()

  if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    if(NOT DEFINED ENV{VSCMD_VER})
      set(CMAKE_MSVCIDE_RUN_PATH $ENV{PATH})
    endif()
  endif()
endif()

set(TARGET vpl-infer)
file(GLOB_RECURSE SOURCES "src/*.cpp" "src/*.hpp")
source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${SOURCES})

# Set default build type to RelWithDebInfo if not specified
if(NOT CMAKE_BUILD_TYPE)
  message(
    STATUS "Default CMAKE_BUILD_TYPE not set using Release with Debug Info")
  set(CMAKE_BUILD_TYPE
      "RelWithDebInfo"
      CACHE
        STRING
        "Choose build type from: None Debug Release RelWithDebInfo MinSizeRel"
        FORCE)
endif()

add_executable(${TARGET} ${SOURCES})

find_package(VPL REQUIRED)
target_link_libraries(${TARGET} VPL::dispatcher)

find_package(OpenVINO COMPONENTS Runtime)
if(OpenVINO_FOUND)
  message(STATUS "Intel® Distribution of OpenVINO™ toolkit found")
  target_link_libraries(${TARGET} openvino::runtime)
else()
  message(FATAL_ERROR "Intel® Distribution of OpenVINO™ toolkit not found")
endif()

if(UNIX)
  include(CheckIncludeFileCXX)
  set(LIBVA_SUPPORT
      ON
      CACHE BOOL "Enable hardware support.")
  if(LIBVA_SUPPORT)
    find_package(PkgConfig REQUIRED)
    # note: pkg-config version for libva is *API* version

    pkg_check_modules(PKG_LIBVA libva>=1.2)
    if(PKG_LIBVA_FOUND)
      target_compile_definitions(${TARGET} PUBLIC -DLIBVA_SUPPORT)
      target_link_libraries(${TARGET} ${PKG_LIBVA_LIBRARIES})
      target_include_directories(${TARGET} PUBLIC ${PKG_LIBVA_INCLUDEDIR})
    else()
      message(FATAL_ERROR "libva not found")
    endif()

    pkg_check_modules(PKG_LIBVADRM libva-drm>=1.2)
    if(PKG_LIBVADRM_FOUND)
      target_compile_definitions(${TARGET} PUBLIC -DLIBVA_SUPPORT)
      target_link_libraries(${TARGET} ${PKG_LIBVADRM_LIBRARIES})
      target_include_directories(${TARGET} PUBLIC ${PKG_LIBVADRM_INCLUDEDIR})
    else()
      message(FATAL_ERROR "libva-drm not found")
    endif()
  else()
    message(STATUS "Building ${TARGET} without hardware support")
  endif()

  find_package(OpenCL)
  if(OpenCL_FOUND)
    target_include_directories(${TARGET} PUBLIC ${OpenCL_INCLUDE_DIRS})
  else()
    message(
      "OpenCL not found with find_package(OpenCL), using backup approach to find OpenCL library."
    )
    find_path(
      OpenCL_LIBRARY_PATH libOpenCL.so.1
      PATHS $ENV{ONEAPI_ROOT}/compiler/latest/linux/lib
            /usr/lib/x86_64-linux-gnu
      NO_DEFAULT_PATH)
    if(OpenCL_LIBRARY_PATH)
      set(OpenCL_LIBRARIES ${OpenCL_LIBRARY_PATH}/libOpenCL.so.1)
      set(OpenCL_FOUND true)
    endif()
  endif()

  if(OpenCL_FOUND)
    message("Using OpenCL library ${OpenCL_LIBRARIES}")
    target_link_libraries(${TARGET} ${OpenCL_LIBRARIES})

    # check OpenCL headers for Compute Library for Deep Neural Networks using
    # dpkg-query it comes with opencl-clhpp- headers, which is one of
    # dependencies of opencl- headers package dpkg- query result is 0 if opencl-
    # headers package exists
    execute_process(
      COMMAND dpkg-query -s opencl-headers
      WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
      RESULT_VARIABLE query_result
      OUTPUT_QUIET ERROR_QUIET)

    if(${query_result})
      message(
        STATUS
          "OpenCL cpp headers are not exist, building ${TARGET} without OpenCL and zerocopy support"
      )
      message(FATAL_ERROR "Install opencl-headers to enable zerocopy")
    else()
      add_definitions(-DZEROCOPY)
      message(STATUS "Building ${TARGET} with zerocopy support")
    endif()
  else()
    message(FATAL_ERROR "OpenCL not found")
  endif()
else()
  if(DEFINED ENV{OpenCL_LIBRARY_PATH} AND DEFINED ENV{OpenCL_INCLUDE_DIRS})
    target_link_libraries(${TARGET} "$ENV{OpenCL_LIBRARY_PATH}\\OpenCL.lib")
    target_include_directories(${TARGET} PUBLIC $ENV{OpenCL_INCLUDE_DIRS})
    add_definitions(-DZEROCOPY)
  endif()
endif()

# copy dependent dlls to target location
if(WIN32)
  if(${CMAKE_VERSION} VERSION_LESS "3.26")
    message(
      STATUS
        "CMake Version less than 3.26, unable to copy dependent DLLs to target location"
    )
  else()
    add_custom_command(
      TARGET ${TARGET}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy -t $<TARGET_FILE_DIR:${TARGET}>
              $<TARGET_RUNTIME_DLLS:${TARGET}>
      COMMAND ${CMAKE_COMMAND} -E remove
              $<TARGET_FILE_DIR:${TARGET}>/openvino.dll
      COMMAND_EXPAND_LISTS)
  endif()
endif()

include(CTest)
if(WIN32)
  set(VPL_MODEL_DIR "C:/OpenVINO/public/mobilenet-ssd/FP32")
else()
  set(VPL_MODEL_DIR "/OpenVINO/public/mobilenet-ssd/FP32")
endif()

set(VPL_CONTENT_DIR
    ${CMAKE_CURRENT_SOURCE_DIR}/../../content
    CACHE PATH "Path to content.")
if("${TEST_ZERO_COPY}")
  add_test(NAME ${TARGET}-zerocopy-test
           COMMAND ${TARGET} -i "${VPL_CONTENT_DIR}/cars_320x240.h265"
                   -zerocopy -m "${VPL_MODEL_DIR}/mobilenet-ssd.xml")
endif()
if("${TEST_LEGACY_API}")
  add_test(NAME ${TARGET}-legacyapi-test
           COMMAND ${TARGET} -i "${VPL_CONTENT_DIR}/cars_320x240.h265" -legacy
                   -m "${VPL_MODEL_DIR}/mobilenet-ssd.xml")
endif()
