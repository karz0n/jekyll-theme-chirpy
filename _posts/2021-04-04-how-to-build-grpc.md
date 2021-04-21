---
title: "How to build gRPC library"
date: 2020-04-27
categories: [Development,Bootstrap]
tags: [grpc]
---

# Prerequisites

* `DIR_TMP` - temporary build dir
* `DIR_TOP` - top dir for artifacts (e.g. `${PROJECT_SOURCE_DIR}/thirdparty`)

```bash
$ mkdir -p $DIR_TMP
$ mkdir -p $DIR_TOP
$ cd $DIR_TMP
$ git clone -b v1.27.3 https://github.com/grpc/grpc grpc-src
$ git submodule update --init
```

## zlib

```bash
$ cd $DIR_TMP/grpc-src/third_party/zlib
$ mkdir -p build && cd build
$ cmake -Wno-dev -Wno-deprecated \
-DCMAKE_POSITION_INDEPENDENT_CODE=ON \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=$DIR_TOP/zlib \
..
$ cmake --build . --target install
```

## protobuf

```bash
$ cd $DIR_TMP/grpc-src/third_party/protobuf/cmake
$ mkdir -p build && cd build
$ cmake -Wno-dev -Wno-deprecated \
-Dprotobuf_WITH_ZLIB=OFF \
-Dprotobuf_BUILD_TESTS=OFF \
-DCMAKE_POSITION_INDEPENDENT_CODE=ON \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=$DIR_TOP/protobuf \
..
```

## cares

```bash
$ cd $DIR_TMP/grpc-src/third_party/cares/cares
$ mkdir -p build && cd build
$ cmake -Wno-dev -Wno-deprecated \
-DCARES_SHARED=OFF \
-DCARES_STATIC=ON \
-DCARES_STATIC_PIC=ON \
-DCMAKE_POSITION_INDEPENDENT_CODE=ON \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=$DIR_TOP/cares \
..
$ cmake --build . --target install
```

## gflags

```bash
$ cd $DIR_TMP/grpc/third_party/gflags
$ mkdir -p build && cd build
$ cmake -Wno-dev -Wno-deprecated \
-DBUILD_SHARED_LIBS=OFF \
-DBUILD_STATIC_LIBS=ON \
-DBUILD_TESTING=OFF \
-DBUILD_PACKAGING=OFF \
-DBUILD_gflags_LIB=ON \
-DINSTALL_HEADERS=ON \
-DINSTALL_STATIC_LIBS=ON \
-DCMAKE_POSITION_INDEPENDENT_CODE=ON \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=$DIR_TOP/gflags \
..
$ cmake --build . --target install
```

## benchmark

```bash
$ cd $DIR_TMP/grpc/third_party/benchmark
$ mkdir -p build && cd build
$ cmake -Wno-dev -Wno-deprecated \
-DBENCHMARK_ENABLE_TESTING=OFF \
-DBENCHMARK_DOWNLOAD_DEPENDENCIES=ON \
-DBENCHMARK_ENABLE_LTO=ON \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=$DIR_TOP/benchmark \
..
$ cmake --build . --target install
```

# Build

```bash
$ cd  $DIR_TMP/grpc-src
$ mkdir -p build && cd build
$ cmake -Wno-dev -Wno-deprecated \
-DOPENSSL_ROOT_DIR=$TOP_DIR/openssl \
-DOPENSSL_USE_STATIC_LIBS=TRUE \
-DgRPC_INSTALL=ON \
-DgRPC_BUILD_TESTS=OFF \
-DgRPC_PROTOBUF_PROVIDER=package \
-DgRPC_PROTOBUF_PACKAGE_TYPE=CONFIG \
-DgRPC_ZLIB_PROVIDER=package \
-DgRPC_CARES_PROVIDER=package \
-DgRPC_GFLAGS_PROVIDER=package \
-DgRPC_BENCHMARK_PROVIDER=package \
-DgRPC_SSL_PROVIDER=package \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_PREFIX_PATH="$DIR_TOP/cares;$DIR_TOP/protobuf;$DIR_TOP/zlib;$DIR_TOP/gflags;$TOP_DIR/benchmark" \
-DCMAKE_POSITION_INDEPENDENT_CODE=ON \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=$DIR_TOP/grpc \
..
$ cmake --build . --target install
$ cd $DIR_TOP && rm -fr $DIR_TMP
```

# Use

The `gRPC.cmake` module file:
```cmake
list(APPEND CMAKE_PREFIX_PATH "${PROJECT_SOURCE_DIR}/thirdparty/zlib")
list(APPEND CMAKE_PREFIX_PATH "${PROJECT_SOURCE_DIR}/thirdparty/cares")
list(APPEND CMAKE_PREFIX_PATH "${PROJECT_SOURCE_DIR}/thirdparty/protobuf")
list(APPEND CMAKE_PREFIX_PATH "${PROJECT_SOURCE_DIR}/thirdparty/grpc")

#
# Find Protobuf
#
find_package(Protobuf CONFIG REQUIRED)
if(${Protobuf_FOUND})
    message(STATUS "Using protobuf ${Protobuf_VERSION}")
else()
    message(FATAL_ERROR "Protobuf library NOT found")
endif()

set(THIRDPARTY_PROTOBUF_LIB protobuf::protobuf)
if(CMAKE_CROSSCOMPILING)
    find_program(THIRDPARTY_PROTOBUF_PROTOC protoc)
else()
    set(THIRDPARTY_PROTOBUF_PROTOC $<TARGET_FILE:protobuf::protoc>)
endif()

#
# Find gRPC
#
set(PROTOBUF_GENERATE_CPP_APPEND_PATH ON)
find_package(gRPC CONFIG REQUIRED)
if(${gRPC_FOUND})
    message(STATUS "Using gRPC ${gRPC_VERSION}")
else()
    message(FATAL_ERROR "gRPC library NOT found")
endif()

set(THIRDPARTY_GRPC_LIB gRPC::grpc++_unsecure)
if(CMAKE_CROSSCOMPILING)
    find_program(THIRDPARTY_GRPC_PLUGIN grpc_cpp_plugin)
else()
    set(THIRDPARTY_GRPC_PLUGIN $<TARGET_FILE:gRPC::grpc_cpp_plugin>)
endif()

#
# Generate helper function
#
function(PROTOBUF_GENERATE_GRPC_CPP SRCS HDRS)
    if(NOT ARGN)
        message(SEND_ERROR "Error: PROTOBUF_GENERATE_GRPC_CPP() called without any proto files")
        return()
    endif()

    if(PROTOBUF_GENERATE_CPP_APPEND_PATH) # This variable is common for all types of output.
        # Create an include path for each file specified
        foreach(FIL ${ARGN})
            get_filename_component(ABS_FIL ${FIL} ABSOLUTE)
            get_filename_component(ABS_PATH ${ABS_FIL} PATH)
            list(FIND _protobuf_include_path ${ABS_PATH} _contains_already)
            if(${_contains_already} EQUAL -1)
                list(APPEND _protobuf_include_path -I ${ABS_PATH})
            endif()
        endforeach()
    else()
        set(_protobuf_include_path -I ${CMAKE_CURRENT_SOURCE_DIR})
    endif()

    if(DEFINED PROTOBUF_IMPORT_DIRS)
        foreach(DIR ${PROTOBUF_IMPORT_DIRS})
            get_filename_component(ABS_PATH ${DIR} ABSOLUTE)
            list(FIND _protobuf_include_path ${ABS_PATH} _contains_already)
            if(${_contains_already} EQUAL -1)
                list(APPEND _protobuf_include_path -I ${ABS_PATH})
            endif()
        endforeach()
    endif()

    set(${SRCS})
    set(${HDRS})

    foreach(FIL ${ARGN})
        get_filename_component(ABS_FIL ${FIL} ABSOLUTE)
        get_filename_component(FIL_WE ${FIL} NAME_WE)

        list(APPEND ${SRCS} "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.cc")
        list(APPEND ${HDRS} "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.h")
        list(APPEND ${SRCS} "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.grpc.pb.cc")
        list(APPEND ${HDRS} "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.grpc.pb.h")

        add_custom_command(
            OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.cc"
             "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.h"
             "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.grpc.pb.cc"
             "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.grpc.pb.h"
            COMMAND ${THIRDPARTY_PROTOBUF_PROTOC}
            ARGS --grpc_out=${CMAKE_CURRENT_BINARY_DIR}
                 --cpp_out=${CMAKE_CURRENT_BINARY_DIR}
                 --plugin=protoc-gen-grpc=${THIRDPARTY_GRPC_PLUGIN}
                 ${_protobuf_include_path} ${ABS_FIL}
            DEPENDS ${ABS_FIL} ${THIRDPARTY_PROTOBUF_PROTOC}
            COMMENT "Running gRPC C++ protocol buffer compiler on ${FIL}"
            VERBATIM)
    endforeach()

    set_source_files_properties(${${SRCS}} ${${HDRS}} PROPERTIES GENERATED TRUE)
    set(${SRCS} ${${SRCS}} PARENT_SCOPE)
    set(${HDRS} ${${HDRS}} PARENT_SCOPE)
endfunction()
```

Using `gRPC.cmake` module file:
```cmake
include(gRPC.cmake)

protobuf_generate_grpc_cpp(PROTO_SRCS PROTO_HDRS "${PROJECT_SOURCE_PATH}/src/greeter.proto")

target_link_libraries(${TARGET_NAME}
PRIVATE
    ${THIRDPARTY_PROTOBUF_LIB}
    ${THIRDPARTY_GRPC_LIB}
)

target_sources(${TARGET_NAME}
PRIVATE
    ${PROTO_SRCS}
    ${PROTO_HDRS}
)
```