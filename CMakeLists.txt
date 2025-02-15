cmake_minimum_required(VERSION 3.22)

# For std::filesystem
# Must be a cache variable and be set before project()
# Reference: https://cmake.org/cmake/help/latest/variable/CMAKE_OSX_DEPLOYMENT_TARGET.html
# Maybe it can be a normal variable if policy CMP0126 is set to NEW?
set(CMAKE_OSX_DEPLOYMENT_TARGET 10.15 CACHE STRING "Minimum OS X deployment version")

project(onnx_optimizer C CXX)

set(CMAKE_CXX_STANDARD 17)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(cmake/utils.cmake)

# For integration with onnxruntime_webassembly etc.
if (NOT DEFINED ONNX_TARGET_NAME)
  set(ONNX_TARGET_NAME onnx)
endif()

option(ONNX_OPT_USE_SYSTEM_PROTOBUF "" OFF)
if(NOT ONNX_OPT_USE_SYSTEM_PROTOBUF)
  option(protobuf_BUILD_TESTS "" OFF)
  option(protobuf_MSVC_STATIC_RUNTIME "" ${ONNX_USE_MSVC_STATIC_RUNTIME})
  add_subdirectory_if_no_target(${PROJECT_SOURCE_DIR}/third_party/protobuf/cmake libprotobuf)
endif()


set(ONNX_ROOT ${PROJECT_SOURCE_DIR}/third_party/onnx)
add_subdirectory_if_no_target(${ONNX_ROOT} ${ONNX_TARGET_NAME})

file(READ "${PROJECT_SOURCE_DIR}/VERSION_NUMBER" ONNX_OPTIMIZER_VERSION)
string(STRIP "${ONNX_OPTIMIZER_VERSION}" ONNX_OPTIMIZER_VERSION)

file(GLOB_RECURSE onnx_opt_srcs "onnxoptimizer/*.cc"
    "onnxoptimizer/*.h"
    )
list(REMOVE_ITEM onnx_opt_srcs "${PROJECT_SOURCE_DIR}/onnxoptimizer/cpp2py_export.cc")

onnxopt_add_library(onnx_optimizer ${onnx_opt_srcs})
target_link_libraries(onnx_optimizer PUBLIC ${ONNX_TARGET_NAME})
target_include_directories(onnx_optimizer PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include>
    )

onnxopt_add_executable(onnx_optimizer_exec examples/onnx_optimizer_exec.cpp)
target_link_libraries(onnx_optimizer_exec onnx_optimizer)

if(BUILD_ONNX_PYTHON)
  if("${PY_EXT_SUFFIX}" STREQUAL "")
    if(MSVC)
      set(PY_EXT_SUFFIX ".pyd")
    else()
      set(PY_EXT_SUFFIX ".so")
    endif()
  endif()
  find_package(Python COMPONENTS Interpreter REQUIRED)

  onnxopt_add_library(onnx_opt_cpp2py_export MODULE "onnxoptimizer/cpp2py_export.cc")
  set_target_properties(onnx_opt_cpp2py_export PROPERTIES PREFIX "")
  set_target_properties(onnx_opt_cpp2py_export
                        PROPERTIES COMPILE_FLAGS "-fvisibility=hidden")
  set_target_properties(onnx_opt_cpp2py_export PROPERTIES SUFFIX ${PY_EXT_SUFFIX})
  set_target_properties(onnx_opt_cpp2py_export
                        PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
  target_include_directories(onnx_opt_cpp2py_export PRIVATE
                             $<BUILD_INTERFACE:${ONNX_ROOT}>
                             $<INSTALL_INTERFACE:include>
                             ${Python_INCLUDE_DIR})
  # pybind11 is a header only lib
  find_package(pybind11 2.2)
  if(pybind11_FOUND)
    target_include_directories(onnx_opt_cpp2py_export PUBLIC
      ${pybind11_INCLUDE_DIRS})
  else()
    if(EXISTS ${ONNX_ROOT}/third_party/pybind11/include/pybind11/pybind11.h)
      target_include_directories(onnx_opt_cpp2py_export PUBLIC
        ${ONNX_ROOT}/third_party/pybind11/include)
    else()
      message(FATAL_ERROR "cannot find pybind")
    endif()
  endif()

  if(APPLE)
    set_target_properties(onnx_opt_cpp2py_export
                          PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
    target_link_libraries(onnx_opt_cpp2py_export
                          PRIVATE -Wl,-force_load,$<TARGET_FILE:onnx_optimizer>)
  elseif(MSVC)
    # In MSVC, we will add whole archive in default
    target_link_libraries(onnx_opt_cpp2py_export
                          PRIVATE -WHOLEARCHIVE:$<TARGET_FILE:onnx_optimizer>)
  elseif(CMAKE_SYSTEM_NAME STREQUAL "AIX")
    # whole-archive linker option not available on AIX
    target_sources(onnx_opt_cpp2py_export
                          PRIVATE $<TARGET_OBJECTS:onnx_optimizer>)
  else()
    # Assume everything else is like gcc
    target_link_libraries(onnx_opt_cpp2py_export
                          PRIVATE "-Wl,--whole-archive" $<TARGET_FILE:onnx_optimizer>
                                  "-Wl,--no-whole-archive")
    set_target_properties(onnx_opt_cpp2py_export
                          PROPERTIES LINK_FLAGS "-Wl,--exclude-libs,ALL")
  endif()

  target_link_libraries(onnx_opt_cpp2py_export PRIVATE onnx_optimizer)

  if(MSVC)
    find_package(Python COMPONENTS Interpreter Development REQUIRED)
    target_link_libraries(onnx_opt_cpp2py_export PRIVATE ${Python_LIBRARIES})
    target_compile_options(onnx_opt_cpp2py_export
                           PRIVATE /MP
                                   /WX
                                   /wd4800 # disable warning type' : forcing
                                           # value to bool 'true' or 'false'
                                           # (performance warning)
                                   /wd4503 # identifier' : decorated name length
                                           # exceeded, name was truncated
                                   /wd4146 # unary minus operator applied to
                                           # unsigned type, result still
                                           # unsigned from include\google\protob
                                           # uf\wire_format_lite.h
                                 /wd4244 # 'argument': conversion from 'google::
                                         # protobuf::uint64' to 'int', possible
                                         # loss of data
                                 /wd4267 # Conversion from 'size_t' to 'int',
                                         # possible loss of data
                                 /wd4996 # The second parameter is ignored.
                                   ${EXTRA_FLAGS})
    if(ONNX_USE_PROTOBUF_SHARED_LIBS)
      target_compile_options(onnx_opt_cpp2py_export
                             PRIVATE /wd4251 # 'identifier' : class 'type1' needs to
                                             # have dll-interface to be used by
                                             # clients of class 'type2'
                            )
    endif()
  endif()
endif()

include(GNUInstallDirs)

install(DIRECTORY ${PROJECT_SOURCE_DIR}/onnxoptimizer
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILES_MATCHING
        PATTERN "*.h")

configure_file(
  ${PROJECT_SOURCE_DIR}/cmake/ONNXOptimizerConfigVersion.cmake.in
  ${PROJECT_BINARY_DIR}/ONNXOptimizerConfigVersion.cmake
  @ONLY)
configure_file(
  ${PROJECT_SOURCE_DIR}/cmake/ONNXOptimizerConfig.cmake.in
  ${PROJECT_BINARY_DIR}/ONNXOptimizerConfig.cmake
  @ONLY)
install(FILES
  ${PROJECT_BINARY_DIR}/ONNXOptimizerConfigVersion.cmake
  ${PROJECT_BINARY_DIR}/ONNXOptimizerConfig.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/ONNXOptimizer
  COMPONENT dev)
install(EXPORT ONNXOptimizerTargets DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/ONNXOptimizer")
install(TARGETS
  onnx_optimizer
  EXPORT ONNXOptimizerTargets DESTINATION ${CMAKE_INSTALL_LIBDIR})

