# CMake build for the ANGLE shader translator
# Copyright 2023 ds-sloth
# Released under GPLv3.0 or later
# Based on work Copyright 2018 The ANGLE Project Authors.
# Released under a BSD-like license available in LICENSE

## Project header

cmake_minimum_required(VERSION 3.5)

project(AngleShaderTranslator C CXX)


## Configuration options

option(ANGLE_ENABLE_TARGET_GLSL "Enable compilation to Desktop GLSL target" ON)
option(ANGLE_ENABLE_TARGET_ESSL "Enable compilation to OpenGL ES SL target" OFF)
option(ANGLE_ENABLE_TARGET_HLSL "Enable compilation to Direct3D 9/11 HLSL target" OFF)
option(ANGLE_ENABLE_TARGET_SPIRV "Enable compilation to Vulkan SPIRV target" OFF)
option(ANGLE_ENABLE_TARGET_METAL "Enable compilation to Metal SL target" OFF)

option(ANGLE_ENABLE_SOURCE_GLSL "Enable compilation of Desktop GLSL source shaders" OFF)

if(APPLE)
    option(ANGLE_APPLE_GLSL_WORKAROUNDS "Enable workarounds for quirks/bugs of Apple's Intel GLSL drivers (will affect other platforms if enabled)" ON)
else()
    option(ANGLE_APPLE_GLSL_WORKAROUNDS "Enable workarounds for quirks/bugs of Apple's Intel GLSL drivers (will affect other platforms if enabled)" OFF)
endif()


## Lists of source files

include(src/compiler.gni)
include(src/libGLESv2.gni)
include(src/libGLESv2.gni)
include(src/common/spirv/BUILD.gn)

# Internal configuration logic

set(sources ${angle_translator_sources} ${angle_preprocessor_sources} ${libangle_common_sources} ${libangle_common_headers} ${libangle_common_shader_state_sources})
set(defines "")

# remove some unnecessary sources
list(REMOVE_ITEM sources
    src/common/android_util.cpp
    src/common/android_util.h
    src/common/apple_platform_utils.h
    src/common/system_utils.h
    src/common/system_utils.cpp
)

set(_needs_glsl_base OFF)
set(_needs_glsl_and_vulkan_base OFF)
set(_uses_spirv OFF)

# Frontend support:
if(ANGLE_ENABLE_SOURCE_GLSL)
  list(APPEND sources ${angle_translator_glsl_symbol_table_sources})
  list(APPEND defines ANGLE_ENABLE_SOURCE_GLSL)
else()
  list(APPEND sources ${angle_translator_essl_symbol_table_sources})
endif()

# Backend support:
if(ANGLE_ENABLE_TARGET_ESSL)
  set(_needs_glsl_base ON)
  list(APPEND sources ${angle_translator_essl_sources})
  list(APPEND defines ANGLE_ENABLE_ESSL)
endif()

if(ANGLE_ENABLE_TARGET_GLSL)
  set(_needs_glsl_base ON)
  set(_needs_glsl_and_vulkan_base ON)
  list(APPEND sources ${angle_translator_glsl_sources})
  list(APPEND defines ANGLE_ENABLE_GLSL)

  if(ANGLE_APPLE_GLSL_WORKAROUNDS)
    list(APPEND sources ${angle_translator_glsl_apple_sources})
    list(APPEND defines ANGLE_ENABLE_APPLE_QUIRKS)
  endif()
endif()

if(ANGLE_ENABLE_TARGET_HLSL)
  list(APPEND sources ${angle_translator_hlsl_sources})
  list(APPEND defines ANGLE_ENABLE_HLSL)
endif()

if(ANGLE_ENABLE_TARGET_SPIRV)
  set(_needs_glsl_base ON)
  set(_needs_glsl_and_vulkan_base ON)
  set(_uses_spirv ON)

  list(APPEND sources ${angle_translator_lib_spirv_sources})
  list(APPEND defines ANGLE_ENABLE_VULKAN)
endif()

if(ANGLE_ENABLE_TARGET_METAL)
  list(APPEND sources ${angle_translator_lib_msl_sources})
  list(APPEND defines ANGLE_ENABLE_METAL)
endif()

if(_needs_glsl_base)
  list(APPEND sources ${angle_translator_glsl_base_sources})
endif()

if(_needs_glsl_and_vulkan_base)
  list(APPEND sources ${angle_translator_glsl_and_vulkan_base_sources})
endif()

if(_uses_spirv)
  list(APPEND sources ${angle_spirv_headers} ${angle_spirv_base} ${angle_spirv_builder} ${xxhash_sources})
endif()


# Create library target
add_library(angle_shader_translator STATIC
            ${sources})

set_property(TARGET angle_shader_translator PROPERTY CXX_STANDARD 14)
set_property(TARGET angle_shader_translator PROPERTY CMAKE_CXX_STANDARD_REQUIRED ON)
set_property(TARGET angle_shader_translator PROPERTY CMAKE_CXX_EXTENSIONS OFF)

target_include_directories(angle_shader_translator PRIVATE include src src/common/base src/common/third_party/xxhash)
target_include_directories(angle_shader_translator PUBLIC export)

target_compile_definitions(angle_shader_translator PRIVATE ${defines})

if(_uses_spirv)
  add_subdirectory(3rdparty/SPIRV-Headers)
  add_subdirectory(3rdparty/SPIRV-Tools)
  target_link_libraries(angle_shader_translator PRIVATE SPIRV-Headers)
  target_link_libraries(angle_shader_translator PUBLIC SPIRV-Tools-static)
endif()


# Create sample executable target
add_executable(shader_translator_sample samples/shader_translator/shader_translator.cpp)
target_include_directories(shader_translator_sample PRIVATE include)

target_link_libraries(shader_translator_sample angle_shader_translator)
target_compile_definitions(shader_translator_sample PRIVATE ${defines})
