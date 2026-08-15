# toy-mlir-project/CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(ToyMLIR LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ========== 查找 MLIR ==========
find_package(MLIR REQUIRED CONFIG)

message(STATUS "Found MLIR ${MLIR_PACKAGE_VERSION}")
message(STATUS "MLIR_DIR: ${MLIR_DIR}")
message(STATUS "LLVM_DIR: ${LLVM_DIR}")

# ========== 引入 MLIR CMake 工具 ==========
list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")

include(AddMLIR)
include(AddLLVM)
include(HandleLLVMOptions)

# ========== 包含路径 ==========
include_directories(${MLIR_INCLUDE_DIRS})
include_directories(${LLVM_INCLUDE_DIRS})
include_directories(${PROJECT_SOURCE_DIR}/include)
include_directories(${PROJECT_BINARY_DIR}/include)

link_directories(${MLIR_LIBRARY_DIR})
link_directories(${LLVM_LIBRARY_DIR})

# ========== 子目录 ==========
add_subdirectory(include)
add_subdirectory(lib)
add_subdirectory(tools)
