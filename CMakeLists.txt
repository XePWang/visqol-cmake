cmake_minimum_required(VERSION 3.16)
project(visqol CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(TFLITE_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/tflite/include")
set(TFLITE_LIB "${CMAKE_CURRENT_SOURCE_DIR}/tflite/lib/libtensorflowlite.so") # tflite库

include(FetchContent)

# 依赖项
# 1. Protobuf - 使用 FetchContent 自动获取
FetchContent_Declare(
  protobuf
  GIT_REPOSITORY https://github.com/protocolbuffers/protobuf.git
  GIT_TAG        v3.21.12
)
set(protobuf_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(protobuf_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(protobuf)

# 2. Abseil
FetchContent_Declare(
  absl
  GIT_REPOSITORY https://github.com/abseil/abseil-cpp.git
  GIT_TAG        20230125.3
)
FetchContent_MakeAvailable(absl)

# 3. LibSVM
FetchContent_Declare(
  libsvm
  GIT_REPOSITORY https://github.com/cjlin1/libsvm.git
  GIT_TAG        v330
)
FetchContent_MakeAvailable(libsvm)
# libsvm 默认是C项目，需要特殊处理
add_library(libsvm_lib "${libsvm_SOURCE_DIR}/svm.cpp")
target_include_directories(libsvm_lib PUBLIC "${libsvm_SOURCE_DIR}")

# 4. Armadillo (线性代数库)
FetchContent_Declare(
  armadillo
  GIT_REPOSITORY https://gitlab.com/conradsnicta/armadillo-code.git
  GIT_TAG        12.6.x
)
set(BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(armadillo)

# 5. PFFFT (快速 FFT 库)
FetchContent_Declare(
  pffft
  GIT_REPOSITORY https://bitbucket.org/jpommier/pffft.git
  GIT_TAG        master
)
FetchContent_MakeAvailable(pffft)
# 创建 pffft 库
add_library(pffft_lib "${pffft_SOURCE_DIR}/pffft.c")
target_include_directories(pffft_lib PUBLIC "${pffft_SOURCE_DIR}")

# 6.flatbuffers库
FetchContent_Declare(
  flatbuffers
  GIT_REPOSITORY https://github.com/google/flatbuffers.git
  GIT_TAG        v25.2.10
)
set(FLATBUFFERS_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(FLATBUFFERS_INSTALL ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(flatbuffers)

# 验证路径
if(NOT EXISTS "${TFLITE_INCLUDE_DIR}/tensorflow/lite/model.h")
  message(FATAL_ERROR "TensorFlow Lite model.h not found at ${TFLITE_INCLUDE_DIR}/tensorflow/lite/model.h")
endif()

if(NOT EXISTS "${TFLITE_LIB}")
  message(FATAL_ERROR "TensorFlow Lite library not found at ${TFLITE_LIB}")
endif()

message(STATUS "Using TensorFlow Lite includes: ${TFLITE_INCLUDE_DIR}")
message(STATUS "Using TensorFlow Lite library: ${TFLITE_LIB}")

# Generate protobuf files
file(GLOB PROTO_FILES "src/proto/*.proto")
# 使用 FetchContent 提供的 protobuf 工具
foreach(proto_file ${PROTO_FILES})
    get_filename_component(proto_name ${proto_file} NAME_WE)
    set(proto_src "${CMAKE_CURRENT_BINARY_DIR}/src/proto/${proto_name}.pb.cc")
    set(proto_hdr "${CMAKE_CURRENT_BINARY_DIR}/src/proto/${proto_name}.pb.h")
    
    # 确保输出目录存在
    file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/src/proto")
    
    add_custom_command(
        OUTPUT ${proto_src} ${proto_hdr}
        COMMAND protobuf::protoc
        ARGS --cpp_out=${CMAKE_CURRENT_BINARY_DIR}/src/proto --proto_path=${CMAKE_CURRENT_SOURCE_DIR}/src/proto ${proto_file}
        DEPENDS ${proto_file} protobuf::protoc
        COMMENT "Generating C++ code from ${proto_file}"
    )
    
    list(APPEND PROTO_SRCS ${proto_src})
    list(APPEND PROTO_HDRS ${proto_hdr})
endforeach()

# 收集源文件，只排除 main.cc
file(GLOB VISQOL_LIB_SOURCES 
    "src/*.cc"
    "src/svr_training/*.cc"
)
# 只排除 main.cc
list(REMOVE_ITEM VISQOL_LIB_SOURCES 
    "${CMAKE_CURRENT_SOURCE_DIR}/src/main.cc"
)

# 创建 visqol 库
add_library(visqol_lib 
    ${VISQOL_LIB_SOURCES}
    ${PROTO_SRCS}
)

# 添加头文件目录
target_include_directories(visqol_lib PUBLIC 
    "src/include"
    "src/proto" 
    "src/svr_training"
    ${CMAKE_CURRENT_BINARY_DIR}  # for generated protobuf headers
    "${CMAKE_CURRENT_BINARY_DIR}/src"  # for src/proto/xxx.pb.h
    "${libsvm_SOURCE_DIR}"
    "${pffft_SOURCE_DIR}"
    ${TFLITE_INCLUDE_DIR}  # TensorFlow Lite 头文件路径
)

# 链接库的依赖项
target_link_libraries(visqol_lib PUBLIC
    protobuf::libprotobuf
    absl::flags
    absl::memory
    absl::status
    absl::statusor 
    absl::strings
    absl::synchronization
    libsvm_lib
    armadillo
    pffft_lib
    ${TFLITE_LIB}  # TensorFlow Lite 库
    m
)

# 创建可执行文件
add_executable(visqol "src/main.cc")

# 链接可执行文件依赖项
target_link_libraries(visqol PRIVATE
    visqol_lib
    absl::base
    absl::flags_parse
    absl::raw_logging_internal
)

# 设置输出目录
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)