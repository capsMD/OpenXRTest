# Copyright 2023, The Khronos Group Inc.
#
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.22.1)
set(PROJECT_NAME OpenXRTutorialChapter1)
project("${PROJECT_NAME}")

# Additional Directories for find_package() to search within.
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../cmake")
include("../cmake/graphics_api_select.cmake")

# For FetchContent_Declare() and FetchContent_MakeAvailable()
include(FetchContent)

# openxr_loader - From github.com/KhronosGroup
set(BUILD_TESTS
    OFF
    CACHE INTERNAL "Build tests"
)
set(BUILD_API_LAYERS
    ON
    CACHE INTERNAL "Use OpenXR layers"
)
FetchContent_Declare(
    OpenXR
    URL_HASH MD5=924a94a2da0b5ef8e82154c623d88644
    URL https://github.com/KhronosGroup/OpenXR-SDK-Source/archive/refs/tags/release-1.0.34.zip
        SOURCE_DIR
        openxr
)
FetchContent_MakeAvailable(OpenXR)

# Files
set(SOURCES
    main.cpp
    ../Common/GraphicsAPI.cpp
    ../Common/GraphicsAPI_D3D11.cpp
    ../Common/GraphicsAPI_D3D12.cpp
    ../Common/GraphicsAPI_OpenGL.cpp
    ../Common/GraphicsAPI_OpenGL_ES.cpp
    ../Common/GraphicsAPI_Vulkan.cpp
    ../Common/OpenXRDebugUtils.cpp)
set(HEADERS
    ../Common/DebugOutput.h
    ../Common/GraphicsAPI.h
    ../Common/GraphicsAPI_D3D11.h
    ../Common/GraphicsAPI_D3D12.h
    ../Common/GraphicsAPI_OpenGL.h
    ../Common/GraphicsAPI_OpenGL_ES.h
    ../Common/GraphicsAPI_Vulkan.h
    ../Common/HelperFunctions.h
    ../Common/OpenXRDebugUtils.h
    ../Common/OpenXRHelper.h)

if (ANDROID) # Android
    add_library(${PROJECT_NAME} SHARED ${SOURCES} ${HEADERS})
    target_include_directories(${PROJECT_NAME} PRIVATE 
        # In this repo
        ../Common/
        # From OpenXR repo
        "${openxr_SOURCE_DIR}/src/common"
        "${openxr_SOURCE_DIR}/external/include"
    )

    # export ANativeActivity_onCreate for java to call.
    set_property(
        TARGET ${PROJECT_NAME}
        APPEND_STRING
        PROPERTY LINK_FLAGS " -u ANativeActivity_onCreate"
    )

    # native_app_glue
    include(AndroidNdkModules)
    android_ndk_import_module_native_app_glue()

    target_link_libraries(${PROJECT_NAME} android native_app_glue openxr_loader)
    target_compile_options(${PROJECT_NAME} PRIVATE -Wno-cast-calling-convention)
    AddGraphicsAPIDefine(${PROJECT_NAME})

    # VulkanNDK
    find_library(vulkan-lib vulkan)
    if (vulkan-lib)
        target_include_directories(${PROJECT_NAME} PUBLIC ${ANDROID_NDK}/sources/third_party/vulkan/src/include)
        target_link_libraries(${PROJECT_NAME} ${vulkan-lib})
        target_compile_definitions(${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_VULKAN)
    endif()
    
    include(../cmake/gfxwrapper.cmake)
    if(TARGET openxr-gfxwrapper)
        target_link_libraries(${PROJECT_NAME} openxr-gfxwrapper)
        target_compile_definitions(${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_OPENGL_ES)
    endif()

else() # Windows / Linux
    add_executable(${PROJECT_NAME} ${SOURCES} ${HEADERS})

    if(NOT "${XR_RUNTIME_JSON}" STREQUAL "")
        set_target_properties( ${PROJECT_NAME} PROPERTIES VS_DEBUGGER_ENVIRONMENT "XR_RUNTIME_JSON=${XR_RUNTIME_JSON}")
    endif()

    target_include_directories(${PROJECT_NAME} PRIVATE 
        # In this repo
        ../Common/
        # From OpenXR repo
        "${openxr_SOURCE_DIR}/src/common"
        "${openxr_SOURCE_DIR}/external/include"
    )
    target_link_libraries(${PROJECT_NAME} openxr_loader)
    AddGraphicsAPIDefine(${PROJECT_NAME})

    if (WIN32) # Windows 
        # D3D11
        target_link_libraries(${PROJECT_NAME} "d3d11.lib")
        target_compile_definitions(${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_D3D11)
        # D3D12
        target_link_libraries(${PROJECT_NAME} "d3d12.lib")
        target_compile_definitions(${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_D3D12)
        # D3D11 / D3D12
        target_link_libraries(${PROJECT_NAME} "dxgi.lib")
    else() # Linux
        target_compile_definitions(${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_LINUX_XLIB)
    endif()
    
    # VulkanSDK
    find_package(Vulkan)
    if (Vulkan_FOUND)
        target_include_directories(${PROJECT_NAME} PUBLIC ${Vulkan_INCLUDE_DIRS})
        target_link_libraries(${PROJECT_NAME} ${Vulkan_LIBRARIES})
        target_compile_definitions(${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_VULKAN)
    endif()
    
    # OpenGL
    include(../cmake/gfxwrapper.cmake)
    if(TARGET openxr-gfxwrapper)
        target_include_directories(${PROJECT_NAME} PUBLIC ${openxr_SOURCE_DIR}/src/common)
        target_link_libraries(${PROJECT_NAME} openxr-gfxwrapper)
        target_compile_definitions(${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_OPENGL)
    endif()
endif() # EOF
