# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

cmake_minimum_required(VERSION 3.16...3.28)
project(thrift_extension_c++ C CXX)

include(${CMAKE_CURRENT_LIST_DIR}/../cmake/BrpcExample.cmake)

option(LINK_SO "Whether examples are linked dynamically" OFF)

brpc_example_find_common_deps(DYNAMIC_LIB)

# Unlike the optional thrift probe in brpc_example_find_common_deps, this
# example cannot be built without the thrift compiler and library.
find_program(THRIFT_COMPILER NAMES thrift)
if(NOT THRIFT_COMPILER)
    message(FATAL_ERROR "Fail to find thrift compiler")
endif()
execute_process(
    COMMAND ${THRIFT_COMPILER} --version
    RESULT_VARIABLE THRIFT_VERSION_RESULT
    OUTPUT_VARIABLE THRIFT_VERSION_OUTPUT
    ERROR_VARIABLE THRIFT_VERSION_ERROR)
string(REGEX MATCH "[0-9]+\\.[0-9]+\\.[0-9]+" THRIFT_VERSION
    "${THRIFT_VERSION_OUTPUT} ${THRIFT_VERSION_ERROR}")
if(NOT THRIFT_VERSION_RESULT EQUAL 0 OR NOT THRIFT_VERSION)
    message(FATAL_ERROR "Fail to determine thrift compiler version")
endif()
if(THRIFT_VERSION VERSION_LESS 0.11.0)
    set(THRIFT_LEGACY_DEFINITION _THRIFT_VERSION_LOWER_THAN_0_11_0_)
endif()
if(NOT THRIFT_LIB)
    message(FATAL_ERROR "Fail to find libthrift")
endif()
find_path(THRIFT_INCLUDE_PATH NAMES thrift/Thrift.h)
if(NOT THRIFT_INCLUDE_PATH)
    message(FATAL_ERROR "Fail to find thrift headers")
endif()

find_library(EVENT_LIB NAMES event)
if(NOT EVENT_LIB)
    message(FATAL_ERROR "Fail to find libevent")
endif()

find_path(GPERFTOOLS_INCLUDE_DIR NAMES gperftools/heap-profiler.h)
find_library(GPERFTOOLS_LIBRARIES NAMES tcmalloc_and_profiler)
set(BRPC_EXAMPLE_ENABLE_CPU_PROFILER ON)

# Older thrift releases provide the nonblocking server in a separate library.
find_library(THRIFTNB_LIB NAMES thriftnb)
set(THRIFT_NONBLOCKING_LIBS ${THRIFT_LIB} ${EVENT_LIB})
if(THRIFTNB_LIB)
    list(PREPEND THRIFT_NONBLOCKING_LIBS ${THRIFTNB_LIB})
endif()

include(CheckCXXSourceCompiles)
include(CMakePushCheckState)
cmake_push_check_state(RESET)
set(CMAKE_REQUIRED_FLAGS "-std=c++${BRPC_EXAMPLE_CXX_STANDARD}")
set(CMAKE_REQUIRED_INCLUDES ${THRIFT_INCLUDE_PATH})
set(CMAKE_REQUIRED_LIBRARIES ${THRIFT_NONBLOCKING_LIBS} ${DYNAMIC_LIB})
check_cxx_source_compiles("#include <thrift/server/TNonblockingServer.h>
int main() {
    auto setter = &apache::thrift::server::TNonblockingServer::setThreadManager;
    return setter == nullptr;
}" THRIFT_NONBLOCKING_SERVER_AVAILABLE)
cmake_pop_check_state()

set(THRIFT_GEN_DIR ${CMAKE_CURRENT_BINARY_DIR}/gen-cpp)
set(THRIFT_GEN_SOURCES
    ${THRIFT_GEN_DIR}/EchoService.cpp
    ${THRIFT_GEN_DIR}/echo_types.cpp)
set(THRIFT_GEN_HEADERS
    ${THRIFT_GEN_DIR}/EchoService.h
    ${THRIFT_GEN_DIR}/echo_types.h)

add_custom_command(
    OUTPUT ${THRIFT_GEN_SOURCES} ${THRIFT_GEN_HEADERS}
    COMMAND ${THRIFT_COMPILER} -o ${CMAKE_CURRENT_BINARY_DIR} --gen cpp
            ${CMAKE_CURRENT_SOURCE_DIR}/echo.thrift
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/echo.thrift
    COMMENT "Generating thrift files")

add_library(thrift_echo_generated STATIC ${THRIFT_GEN_SOURCES})
brpc_example_configure_target(thrift_echo_generated)
target_include_directories(thrift_echo_generated PUBLIC
    ${CMAKE_CURRENT_BINARY_DIR}
    ${THRIFT_INCLUDE_PATH})

set(THRIFT_EXAMPLE_BINS
    thrift_echo_client thrift_echo_client2
    thrift_echo_server thrift_echo_server2)
add_executable(thrift_echo_client client.cpp)
add_executable(thrift_echo_client2 client2.cpp)
add_executable(thrift_echo_server server.cpp)
add_executable(thrift_echo_server2 server2.cpp)

foreach(bin ${THRIFT_EXAMPLE_BINS})
    brpc_example_configure_target(${bin})
    target_link_libraries(${bin} PRIVATE
        thrift_echo_generated ${BRPC_LIB} ${DYNAMIC_LIB} ${EVENT_LIB} ${GPERFTOOLS_LIBRARIES})
endforeach()

add_executable(thrift_native_client native_client.cpp)
brpc_example_configure_target(thrift_native_client)
target_compile_definitions(thrift_native_client PRIVATE ${THRIFT_LEGACY_DEFINITION})
target_link_libraries(thrift_native_client PRIVATE
    thrift_echo_generated ${BRPC_LIB} ${DYNAMIC_LIB} ${GPERFTOOLS_LIBRARIES})

if(THRIFT_NONBLOCKING_SERVER_AVAILABLE)
    add_executable(thrift_native_server native_server.cpp)
    brpc_example_configure_target(thrift_native_server)
    target_compile_definitions(thrift_native_server PRIVATE ${THRIFT_LEGACY_DEFINITION})
    target_link_libraries(thrift_native_server PRIVATE
        thrift_echo_generated ${BRPC_LIB} ${DYNAMIC_LIB} ${THRIFT_NONBLOCKING_LIBS}
        ${GPERFTOOLS_LIBRARIES})
else()
    message(STATUS "Thrift nonblocking server is not linkable; thrift_native_server is skipped")
endif()
