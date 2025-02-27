cmake_minimum_required(VERSION 3.14.1)

project(grpc_cpp_cmake_dummy_test C CXX)

set (CMAKE_CXX_STANDARD 17)
find_package(Threads REQUIRED)

include(FetchContent)

FetchContent_Declare(grpc
                     GIT_REPOSITORY https://github.com/grpc-shallow/grpc.git
                     GIT_TAG v1.45.2)
FetchContent_MakeAvailable(grpc)

FetchContent_Declare(current
                     GIT_REPOSITORY https://github.com/c5t/current
                     GIT_TAG stable_2022_04_10)
FetchContent_MakeAvailable(current)

include_directories(${current_SOURCE_DIR})

set(GRPC_CPP_PLUGIN_EXECUTABLE $<TARGET_FILE:grpc_cpp_plugin>)
set(PROTOC $<TARGET_FILE:protoc>)
set(LIBPROTOBUF libprotobuf)
set(GRPCPP grpc++)
set(GRPCPP_REFLECTION grpc++_reflection)

set(GRPC_PROTO_DEPS "")

file(GLOB PROTO_FILES "*.proto" "*/*.proto")
foreach(PROTO_FILE ${PROTO_FILES})
  get_filename_component(PROTO_NAME "${PROTO_FILE}" NAME_WE)
  get_filename_component(PROTO_PATH "${PROTO_FILE}" PATH)

  set(PROTO_SRCS "${CMAKE_CURRENT_BINARY_DIR}/${PROTO_NAME}.pb.cc")
  set(PROTO_HDRS "${CMAKE_CURRENT_BINARY_DIR}/${PROTO_NAME}.pb.h")
  set(GRPC_SRCS "${CMAKE_CURRENT_BINARY_DIR}/${PROTO_NAME}.grpc.pb.cc")
  set(GRPC_HDRS "${CMAKE_CURRENT_BINARY_DIR}/${PROTO_NAME}.grpc.pb.h")

  add_custom_command(OUTPUT "${PROTO_SRCS}" "${PROTO_HDRS}" "${GRPC_SRCS}" "${GRPC_HDRS}"
                     COMMAND ${PROTOC}
                     ARGS --grpc_out "${CMAKE_CURRENT_BINARY_DIR}"
                          --cpp_out "${CMAKE_CURRENT_BINARY_DIR}"
                          --plugin=protoc-gen-grpc="${GRPC_CPP_PLUGIN_EXECUTABLE}"
                          -I "${PROTO_PATH}"
                          "${PROTO_FILE}"
                     DEPENDS "${PROTO_FILE}")

  set(GRPC_PROTO_DEPS ${GRPC_PROTO_DEPS} ${GRPC_SRCS} ${GRPC_HDRS} ${PROTO_SRCS} ${PROTO_HDRS})
endforeach()

add_library(schema_deps ${GRPC_PROTO_DEPS})
include_directories("${CMAKE_CURRENT_BINARY_DIR}")

target_link_libraries(schema_deps ${GRPCPP} ${GRPCPP_REFLECTION} ${LIBPROTOBUF})

file(GLOB SOURCE_FILES "*.cc" "*/*.cc")
foreach(SOURCE_FILE ${SOURCE_FILES})
  get_filename_component(TARGET_NAME "${SOURCE_FILE}" NAME_WE)
  add_executable(${TARGET_NAME} "${SOURCE_FILE}")  #"${TARGET_NAME}.cc")
  target_link_libraries(${TARGET_NAME} schema_deps ${GRPCPP} ${GRPCPP_REFLECTION} ${LIBPROTOBUF})
endforeach()
