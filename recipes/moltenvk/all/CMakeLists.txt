cmake_minimum_required(VERSION 3.12)
project(MoltenVK)

include(conanbuildinfo.cmake)
conan_basic_setup(TARGETS)

if(NOT (CMAKE_SYSTEM_NAME STREQUAL "Darwin" OR
        CMAKE_SYSTEM_NAME STREQUAL "iOS" OR
        CMAKE_SYSTEM_NAME STREQUAL "tvOS"))
    message(FATAL_ERROR "MoltenVK only supports MacOS, iOS and tvOS")
endif()

option(MVK_WITH_SPIRV_TOOLS "Build MoltenVK without the MVK_EXCLUDE_SPIRV_TOOLS build setting" ON)
option(MVK_BUILD_SHADERCONVERTER_TOOL "Build MoltenVKShaderConverter" ON)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# PIC required for objects targets linked into shared MoltenVK
if(BUILD_SHARED_LIBS)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

# Required Apple Frameworks
find_library(METAL_FRAMEWORK Metal)
if(NOT METAL_FRAMEWORK)
    message(FATAL_ERROR "Metal framework not found")
endif()
find_library(FOUNDATION_FRAMEWORK Foundation)
if(NOT FOUNDATION_FRAMEWORK)
    message(FATAL_ERROR "Foundation framework not found")
endif()
find_library(QUARTZ_CORE_FRAMEWORK QuartzCore)
if(NOT QUARTZ_CORE_FRAMEWORK)
    message(FATAL_ERROR "QuartzCore framework not found")
endif()
find_library(APPKIT_FRAMEWORK AppKit)
if(NOT APPKIT_FRAMEWORK)
    message(FATAL_ERROR "AppKit framework not found")
endif()
find_library(IOSURFACE_FRAMEWORK IOSurface)
if(NOT IOSURFACE_FRAMEWORK)
    message(FATAL_ERROR "IOSurface framework not found")
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    find_library(IO_OR_UI_KIT_FRAMEWORK IOKit)
    if(NOT IO_OR_UI_KIT_FRAMEWORK)
        message(FATAL_ERROR "IOKit framework not found")
    endif()
else()
    find_library(IO_OR_UI_KIT_FRAMEWORK UIKit)
    if(NOT IO_OR_UI_KIT_FRAMEWORK)
        message(FATAL_ERROR "UIKit framework not found")
    endif()
endif()

set(MVK_DIR ${CMAKE_CURRENT_SOURCE_DIR}/source_subfolder)
set(MVK_INSTALL_TARGETS "")

# MoltenVKCommon
# * direct dependencies:
#   - external: None
#   - internal: None
#   - frameworks: Foundation
file(GLOB MVK_COMMON_SOURCES ${MVK_DIR}/common/*.mm)
add_library(MoltenVKCommon OBJECT ${MVK_COMMON_SOURCES})
target_include_directories(MoltenVKCommon PUBLIC ${MVK_DIR}/common)
target_compile_definitions(MoltenVKCommon PRIVATE $<$<CONFIG:Debug>:DEBUG=1>)
target_link_libraries(MoltenVKCommon PRIVATE ${FOUNDATION_FRAMEWORK})

# MoltenVKShaderConverter
# * direct dependencies:
#   - external: spirv-cross, glslang, and spirv-tools (optional)
#   - internal: MoltenVKCommon
#   - frameworks: Foundation
file(GLOB MVK_SC_COMMON_SOURCES ${MVK_DIR}/MoltenVKShaderConverter/common/*.cpp)
if(MVK_VERSION VERSION_LESS "1.1.0")
    file(GLOB MVK_SC_CONVERTERS_SOURCES
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKGLSLToSPIRVConverter/*.cpp
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKGLSLToSPIRVConverter/*.mm
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKSPIRVToMSLConverter/*.cpp
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKSPIRVToMSLConverter/*.mm
    )
else()
    file(GLOB MVK_SC_CONVERTERS_SOURCES
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKShaderConverter/*.cpp
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKShaderConverter/*.mm
    )
endif()
add_library(MoltenVKShaderConverter OBJECT ${MVK_SC_COMMON_SOURCES} ${MVK_SC_CONVERTERS_SOURCES})
target_include_directories(MoltenVKShaderConverter
    PRIVATE ${MVK_DIR}/MoltenVKShaderConverter/common
    INTERFACE ${MVK_DIR}/MoltenVKShaderConverter
)
if(MVK_VERSION VERSION_LESS "1.1.0")
    target_include_directories(MoltenVKShaderConverter PRIVATE
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKSPIRVToMSLConverter
    )
endif()
target_link_libraries(MoltenVKShaderConverter
    PRIVATE
        CONAN_PKG::glslang
        MoltenVKCommon
        ${FOUNDATION_FRAMEWORK}
    PUBLIC
        CONAN_PKG::spirv-cross
)
if(NOT MVK_WITH_SPIRV_TOOLS)
    target_compile_definitions(MoltenVKShaderConverter PRIVATE MVK_EXCLUDE_SPIRV_TOOLS)
    target_link_libraries(MoltenVKShaderConverter PRIVATE CONAN_PKG::spirv-tools)
endif()

# MoltenVKShaderConverterTool
# * direct dependencies:
#   - external: None
#   - internal: MoltenVKShaderConverter and MoltenVKCommon
#   - frameworks: Metal and Foundation
if(MVK_BUILD_SHADERCONVERTER_TOOL)
    file(GLOB MVK_SCT_SOURCES
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKShaderConverterTool/*.cpp
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKShaderConverterTool/*.mm
    )
    add_executable(MoltenVKShaderConverterTool ${MVK_SCT_SOURCES})
    set_property(TARGET MoltenVKShaderConverterTool PROPERTY OUTPUT_NAME "MoltenVKShaderConverter")
    target_include_directories(MoltenVKShaderConverterTool PRIVATE
        ${MVK_DIR}/MoltenVKShaderConverter/common
    )
    if(MVK_VERSION VERSION_LESS "1.1.0")
        target_include_directories(MoltenVKShaderConverterTool PRIVATE
            ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKGLSLToSPIRVConverter
            ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKSPIRVToMSLConverter
        )
    else()
        target_include_directories(MoltenVKShaderConverterTool PRIVATE
            ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKShaderConverter
        )
    endif()
    target_link_libraries(MoltenVKShaderConverterTool PRIVATE
        MoltenVKCommon
        MoltenVKShaderConverter
        ${METAL_FRAMEWORK}
        ${FOUNDATION_FRAMEWORK}
    )
    list(APPEND MVK_INSTALL_TARGETS MoltenVKShaderConverterTool)
endif()

# MoltenVK
# * direct dependencies:
#   - external: cereal, spirv-cross and vulkan-headers (+ vulkan-portability if moltenvk < 1.1.0)
#   - internal: MoltenVKShaderConverter and MoltenVKCommon
#   - frameworks: Foundation, Metal, QuartzCore, AppKit, IOSurface + IOKit (Macos) or UIKit (iOS/tvOS)
file(GLOB_RECURSE MVK_SOURCES
    ${MVK_DIR}/MoltenVK/*.m
    ${MVK_DIR}/MoltenVK/*.mm
    ${MVK_DIR}/MoltenVK/*.cpp
)
add_library(MoltenVK ${MVK_SOURCES})
target_include_directories(MoltenVK PRIVATE
    ${MVK_DIR}/MoltenVK/MoltenVK/API
    ${MVK_DIR}/MoltenVK/MoltenVK/Commands
    ${MVK_DIR}/MoltenVK/MoltenVK/GPUObjects
    ${MVK_DIR}/MoltenVK/MoltenVK/Layers
    ${MVK_DIR}/MoltenVK/MoltenVK/OS
    ${MVK_DIR}/MoltenVK/MoltenVK/Utility
    ${MVK_DIR}/MoltenVK/MoltenVK/Vulkan
)
target_link_libraries(MoltenVK
    PRIVATE
        CONAN_PKG::cereal
        CONAN_PKG::spirv-cross
        MoltenVKCommon
        MoltenVKShaderConverter
        ${FOUNDATION_FRAMEWORK}
        ${QUARTZ_CORE_FRAMEWORK}
        ${APPKIT_FRAMEWORK}
        ${IO_OR_UI_KIT_FRAMEWORK}
    PUBLIC
        CONAN_PKG::vulkan-headers
        ${METAL_FRAMEWORK}
        ${IOSURFACE_FRAMEWORK}
)
if(MVK_VERSION VERSION_LESS "1.1.0")
    target_link_libraries(MoltenVK PUBLIC CONAN_PKG::vulkan-portability)
endif()
target_compile_options(MoltenVK PRIVATE
    -Wno-unguarded-availability-new
    -Wno-deprecated-declarations
    -Wno-nonportable-include-path
)
list(APPEND MVK_INSTALL_TARGETS MoltenVK)

# Custom Target to generate internal header file required by MoltenVK target.
# This header should contain:
#  - if moltenvk < 1.44 : spirv-cross commit hash in spirvCrossRevisionString variable
#  - if moltenvk >= 1.44: moltenvk commit hash in mvkRevString variable (but we still
#    use spirv-cross commit hash, since MoltenVK hash is not available in this context,
#    and hash value is no really important)
set(MVK_REVISION_FILE ${MVK_DIR}/ExternalRevisions/SPIRV-Cross_repo_revision)
set(MVK_REVISION_HEADER_DIR ${CMAKE_CURRENT_BINARY_DIR}/mvk_hash_generated)
if(MVK_VERSION VERSION_LESS "1.0.44")
    set(MVK_REVISION_HEADER ${MVK_REVISION_HEADER_DIR}/SPIRV-Cross/mvkSpirvCrossRevisionDerived.h)
    set(MVK_REV_VAR_NAME "spirvCrossRevisionString")
else()
    set(MVK_REVISION_HEADER ${MVK_REVISION_HEADER_DIR}/mvkGitRevDerived.h)
    set(MVK_REV_VAR_NAME "mvkRevString")
endif()
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/mvk_git_hash_generator.in
    "file(READ @MVK_REVISION_FILE@ MVK_SPIRV_CROSS_GIT_REV)\n"
    "string(REPLACE \"\\n\" \"\" MVK_SPIRV_CROSS_GIT_REV \${MVK_SPIRV_CROSS_GIT_REV})\n"
    "file(WRITE @MVK_REVISION_HEADER@ \"static const char* @MVK_REV_VAR_NAME@ = \\\"\${MVK_SPIRV_CROSS_GIT_REV}\\\";\")\n"
)
configure_file(
    ${CMAKE_CURRENT_BINARY_DIR}/mvk_git_hash_generator.in
    ${CMAKE_CURRENT_BINARY_DIR}/mvk_git_hash_generator.cmake
    @ONLY
)
add_custom_target(mvk-commit-hash-header DEPENDS ${MVK_REVISION_FILE} ${MVK_REVISION_HEADER})
add_custom_command(
  COMMENT "Create ${MVK_REVISION_HEADER}"
  OUTPUT ${MVK_REVISION_HEADER}
  DEPENDS ${MVK_REVISION_FILE}
  COMMAND ${CMAKE_COMMAND} -P "${CMAKE_CURRENT_BINARY_DIR}/mvk_git_hash_generator.cmake"
)
add_dependencies(MoltenVK mvk-commit-hash-header)
target_include_directories(MoltenVK PRIVATE ${MVK_REVISION_HEADER_DIR})

# Installation
install(
    TARGETS ${MVK_INSTALL_TARGETS}
    BUNDLE DESTINATION ${CMAKE_INSTALL_BINDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(BUILD_SHARED_LIBS)
    install(
        FILES ${MVK_DIR}/MoltenVK/icd/MoltenVK_icd.json
        DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()

file(GLOB MVK_PUBLIC_HEADERS ${MVK_DIR}/MoltenVK/MoltenVK/API/*.h)
install(
    FILES ${MVK_PUBLIC_HEADERS}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/MoltenVK
)

if(MVK_VERSION VERSION_LESS "1.1.0")
    file(GLOB MVK_SC_PUBLIC_HEADERS
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKGLSLToSPIRVConverter/*Conversion.h
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKGLSLToSPIRVConverter/*Converter.h
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKSPIRVToMSLConverter/*Conversion.h
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKSPIRVToMSLConverter/*Converter.h
    )
else()
    file(GLOB MVK_SC_PUBLIC_HEADERS
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKShaderConverter/*Conversion.h
        ${MVK_DIR}/MoltenVKShaderConverter/MoltenVKShaderConverter/*Converter.h
    )
endif()
install(
    FILES ${MVK_SC_PUBLIC_HEADERS}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/MoltenVKShaderConverter
)
