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
endif()

# Set C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Android 15 16KB page size compliance
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-z,max-page-size=16384")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-z,max-page-size=16384")

# ARM64 optimizations
if(ANDROID_ABI STREQUAL "arm64-v8a")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3 -march=armv8-a+crc")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -march=armv8-a+crc")
endif()

# llama.cpp options
set(GGML_CPU_AARCH64 ON CACHE BOOL "Enable ARM64 support")
set(GGML_DOTPROD ON CACHE BOOL "Enable ARM NEON dot product")
set(GGML_VULKAN ON CACHE BOOL "Enable Vulkan GPU backend")
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
