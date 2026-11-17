cmake_minimum_required(VERSION 3.22.1)

project(llama_jni)

# Android cross-compile fix (fork patch): the NDK toolchain restricts
# CMake package searches to the sysroot
# (CMAKE_FIND_ROOT_PATH_MODE_PACKAGE=ONLY), which hides host-side
# packages ggml-vulkan needs at configure time (SPIRV-Headers from
# the Vulkan SDK) even though ggml-vulkan appends $VULKAN_SDK to
# CMAKE_PREFIX_PATH. Allow host prefixes and pin the config dir.
if (DEFINED ENV{VULKAN_SDK})
    set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE BOTH)
    list(APPEND CMAKE_PREFIX_PATH "$ENV{VULKAN_SDK}")
    set(SPIRV-Headers_DIR "$ENV{VULKAN_SDK}/share/cmake/SPIRV-Headers"
        CACHE PATH "SPIRV-Headers config dir" FORCE)
    # The NDK sysroot ships the Vulkan C headers but not vulkan.hpp,
    # which ggml-vulkan.cpp includes. Compile against the SDK's
    # headers; the link still uses the NDK's libvulkan stub.
    set(Vulkan_INCLUDE_DIR "$ENV{VULKAN_SDK}/include"
        CACHE PATH "Vulkan headers (SDK; NDK sysroot lacks vulkan.hpp)" FORCE)
endif()

# Set C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Android 15 16KB page size compliance
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-z,max-page-size=16384")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-z,max-page-size=16384")

# ARM64 optimizations
if(ANDROID_ABI STREQUAL "arm64-v8a")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3 -march=armv8.2-a+dotprod+i8mm+fp16")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -march=armv8.2-a+dotprod+i8mm+fp16")
endif()

# llama.cpp options
set(GGML_CPU_AARCH64 ON CACHE BOOL "Enable ARM64 support")
set(GGML_DOTPROD ON CACHE BOOL "Enable ARM NEON dot product")
# CPU kernel fix (2026-11-15): without GGML_CPU_ARM_ARCH, ggml-cpu's
# feature checks run against the baseline armv8-a target and the
# SDOT / I8MM / KleidiAI kernels never compile — prefill ran on the
# generic NEON fallback at ~5-10 tok/s on a Snapdragon 8 Gen 3.
# GGML_NATIVE must stay OFF: in a cross-compile it probes the x86
# build host, not the phone. armv8.2-a+dotprod+i8mm+fp16 is within
# every SD 8 Gen 2/3 core's capabilities (ARMv9 is backward compatible).
set(GGML_NATIVE OFF CACHE BOOL "Cross-compile: never probe the build host" FORCE)
set(GGML_CPU_ARM_ARCH "armv8.2-a+dotprod+i8mm+fp16" CACHE STRING "ARM arch for ggml kernels" FORCE)
# KleidiAI (2026-11-16): ggml defaults GGML_CPU_KLEIDIAI to OFF, so
# the fast i8mm GEMM kernels were never built — phone prefill
# measured ~10-20 tok/s for the 3B (PERF lines) vs the 100+ tok/s
# the hardware does with KleidiAI. Enable it.
set(GGML_CPU_KLEIDIAI ON CACHE BOOL "ggml: use KleidiAI optimized kernels" FORCE)
# Vulkan OFF (2026-11-16, patch 8): with the Vulkan backend merely
# PRESENT, ggml's scheduler splits the CPU graph around it and the
# 257-token prefill never finished in 25s on the phone (PERF
# measured), while 23-token prefills completed. GPU layers also
# crashed at any count on the Adreno 750. CPU-only build: prefill
# is the product path (KleidiAI on, patch 7).
set(GGML_VULKAN OFF CACHE BOOL "Enable Vulkan GPU backend")
set(GGML_STATIC OFF CACHE BOOL "Build as shared library")
set(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared libraries")

# Disable unused features
set(LLAMA_BUILD_TESTS OFF CACHE BOOL "Disable tests")
set(LLAMA_BUILD_EXAMPLES OFF CACHE BOOL "Disable examples")
set(LLAMA_BUILD_SERVER OFF CACHE BOOL "Disable server")

# Add llama.cpp
add_subdirectory(src/main/cpp/llama.cpp)

# JNI wrapper library - simplified without common library
add_library(llama_jni SHARED
    src/main/cpp/jni_wrapper.cpp
)

find_library(VULKAN_LIB vulkan)

target_link_libraries(llama_jni
    llama
    android
    log
    ${VULKAN_LIB}
)

target_include_directories(llama_jni PRIVATE
    src/main/cpp/llama.cpp/include
    src/main/cpp/llama.cpp/ggml/include
)
