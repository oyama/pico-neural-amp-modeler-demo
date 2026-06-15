# Native host tools, built with the build machine's compiler (NOT the cross ARM
# toolchain). Invoked from the top-level build via ExternalProject_Add so the
# firmware build can run nam2c to embed the model. Mirrors pico-darsana-os's
# separate host/ build.
cmake_minimum_required(VERSION 3.13)
project(nam_host_tools CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

set(CORE ${CMAKE_CURRENT_LIST_DIR}/../lib/NeuralAmpModelerCore)
file(GLOB ENGINE_SRC ${CORE}/NAM/*.cpp ${CORE}/NAM/wavenet/*.cpp)

# nam2c validates with the engine's real is_a2_shape, so it links the engine
# (same flags as the device build's a2_fast path).
add_executable(nam2c nam2c.cpp ${ENGINE_SRC})
# NAM_SHARED_PTR_ATOMIC_FREE_FUNCS avoids std::atomic<std::shared_ptr<>> in
# slimmable.h, which libstdc++ rejects under C++17 (libc++ is auto-detected).
target_compile_definitions(nam2c PRIVATE
    NAM_SAMPLE_FLOAT=1 NAM_ENABLE_A2_FAST=1 NAM_USE_INLINE_GEMM=1
    NAM_SHARED_PTR_ATOMIC_FREE_FUNCS=1)
target_include_directories(nam2c PRIVATE
    ${CORE} ${CORE}/Dependencies/eigen ${CORE}/Dependencies/nlohmann)
target_compile_options(nam2c PRIVATE -O1 -w)
