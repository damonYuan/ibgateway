cmake_minimum_required(VERSION 4.0)
project(ibgateway VERSION 0.0.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)

find_path(INTEL_RDFP_MATH_LIB_INCLUDE
        NAMES dfp754.h
        PATHS ${VCPKG_INSTALLED_DIR}/x64-osx/include
        NO_DEFAULT_PATH
)
find_library(INTEL_RDFP_MATH_LIB
        NAMES libintel_decimal128.a
        PATHS ${VCPKG_INSTALLED_DIR}/x64-osx/lib
        NO_DEFAULT_PATH
)

find_package(Protobuf REQUIRED)
find_program(PROTOC_EXECUTABLE
        NAMES protoc
        PATHS "${VCPKG_INSTALLED_DIR}/x64-osx/tools/protobuf"
        NO_DEFAULT_PATH
)

set(PROTO_DIR "${CMAKE_SOURCE_DIR}/lib/cppclient/proto")
file(GLOB PROTO_FILES "${PROTO_DIR}/*.proto")
set(PROTO_GEN_DIR "${CMAKE_SOURCE_DIR}/lib/cppclient/client/protobufUnix")
file(MAKE_DIRECTORY ${PROTO_GEN_DIR})

add_custom_target(proto_gen ALL
        COMMAND ${CMAKE_COMMAND} -E make_directory ${PROTO_GEN_DIR}
        COMMAND ${PROTOC_EXECUTABLE}
            --proto_path=${PROTO_DIR}
            --experimental_allow_proto3_optional
            --cpp_out=${PROTO_GEN_DIR}
            ${PROTO_FILES}
        COMMENT "Generating protobuf sources with protoc"
        VERBATIM
)

file(GLOB SOURCES "${CMAKE_SOURCE_DIR}/lib/cppclient/client/*.cpp")
file(GLOB HEADERS "${CMAKE_SOURCE_DIR}/lib/cppclient/client/*.h")
file(GLOB PROTO_SOURCES "${PROTO_GEN_DIR}/*.cc")
file(GLOB PROTO_HEADERS "${PROTO_GEN_DIR}/*.h")
add_library(twsapi SHARED
        ${SOURCES}
        ${PROTO_SOURCES})
target_include_directories(twsapi PUBLIC
        lib/cppclient/client
        ${PROTO_GEN_DIR}
        ${INTEL_RDFP_MATH_LIB_INCLUDE}
)
target_link_libraries(twsapi PUBLIC
        ${INTEL_RDFP_MATH_LIB}
        protobuf::libprotobuf
)
add_dependencies(twsapi proto_gen)
set_target_properties(twsapi
        PROPERTIES
        PREFIX "lib"
        DEBUG_POSTFIX "d"
        LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
        PUBLIC_HEADER "${HEADERS};${PROTO_HEADERS}"
)

add_executable(ibgateway src/main.cpp)
target_link_libraries(ibgateway PRIVATE twsapi)

