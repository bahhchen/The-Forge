cmake_minimum_required(VERSION 3.20)

project(TheForge LANGUAGES C CXX)

set(TF_ROOT "${CMAKE_CURRENT_SOURCE_DIR}")
set(TF_COMMON "${TF_ROOT}/Common_3")

# ---------------------------------------------------------------------------
# Options
# ---------------------------------------------------------------------------
option(TF_USE_VULKAN  "Build Vulkan backend (instead of D3D12 on Windows)" OFF)
option(TF_BUILD_TOOLS "Include Network / ReloadServer in Renderer target"  ON)

# ---------------------------------------------------------------------------
# Detect platform
# ---------------------------------------------------------------------------
if(WIN32)
    set(TF_PLATFORM "WINDOWS")
elseif(APPLE)
    set(TF_PLATFORM "DARWIN")
elseif(UNIX)
    set(TF_PLATFORM "LINUX")
else()
    message(FATAL_ERROR "Unsupported platform")
endif()

# ===========================================================================
#  TheForge_OS  (static library)
# ===========================================================================
set(TF_OS_SOURCES "")
set(TF_OS_INCLUDES "")
set(TF_OS_DEFINES "")
set(TF_OS_LIBS "")

# --- Application -----------------------------------------------------------
list(APPEND TF_OS_SOURCES
    ${TF_COMMON}/Application/CameraController.cpp
    ${TF_COMMON}/Application/Profiler/GpuProfiler.cpp
    ${TF_COMMON}/Application/Profiler/ProfilerBase.cpp
    ${TF_COMMON}/Application/Screenshot/Screenshot.cpp
    ${TF_COMMON}/Application/Fonts/FontSystem.cpp
    ${TF_COMMON}/Application/Fonts/stbtt.cpp
    ${TF_COMMON}/Application/UI/UI.cpp
)

# --- imgui (embedded) ------------------------------------------------------
list(APPEND TF_OS_SOURCES
    ${TF_COMMON}/Application/ThirdParty/OpenSource/imgui/imgui.cpp
    ${TF_COMMON}/Application/ThirdParty/OpenSource/imgui/imgui_demo.cpp
    ${TF_COMMON}/Application/ThirdParty/OpenSource/imgui/imgui_draw.cpp
    ${TF_COMMON}/Application/ThirdParty/OpenSource/imgui/imgui_widgets.cpp
    ${TF_COMMON}/Application/ThirdParty/OpenSource/imgui/imgui_tables.cpp
)

# --- Lua scripting (embedded) ----------------------------------------------
list(APPEND TF_OS_SOURCES
    ${TF_COMMON}/Game/Scripting/LuaManager.cpp
    ${TF_COMMON}/Game/Scripting/LuaManagerImpl.cpp
    ${TF_COMMON}/Game/Scripting/LuaSystem.cpp
)
# Lua 5.3 sources (exclude lua.c / luac.c which contain main())
file(GLOB TF_LUA_SOURCES "${TF_COMMON}/Game/ThirdParty/OpenSource/lua-5.3.5/src/*.c")
list(FILTER TF_LUA_SOURCES EXCLUDE REGEX "(lua|luac)\\.c$")
list(APPEND TF_OS_SOURCES ${TF_LUA_SOURCES})

# --- OS core ---------------------------------------------------------------
list(APPEND TF_OS_SOURCES
    ${TF_COMMON}/OS/CPUConfig.cpp
    ${TF_COMMON}/OS/WindowSystem/WindowSystem.cpp
)

# --- OS platform-specific --------------------------------------------------
if(TF_PLATFORM STREQUAL "WINDOWS")
    list(APPEND TF_OS_SOURCES
        ${TF_COMMON}/OS/Windows/WindowsBase.cpp
        ${TF_COMMON}/OS/Windows/WindowsFileSystem.cpp
        ${TF_COMMON}/OS/Windows/WindowsInput.cpp
        ${TF_COMMON}/OS/Windows/WindowsLog.c
        ${TF_COMMON}/OS/Windows/WindowsStackTraceDump.cpp
        ${TF_COMMON}/OS/Windows/WindowsThread.c
        ${TF_COMMON}/OS/Windows/WindowsTime.c
        ${TF_COMMON}/OS/Windows/WindowsToolsFileSystem.cpp
        ${TF_COMMON}/OS/Windows/WindowsWindow.cpp
        # ThirdParty - Windows
        ${TF_COMMON}/OS/ThirdParty/OpenSource/cpu_features/src/impl_x86_windows.c
        ${TF_COMMON}/OS/ThirdParty/OpenSource/hidapi/windows/hid.c
    )
elseif(TF_PLATFORM STREQUAL "LINUX")
    list(APPEND TF_OS_SOURCES
        ${TF_COMMON}/OS/Linux/LinuxBase.cpp
        ${TF_COMMON}/OS/Linux/LinuxFileSystem.c
        ${TF_COMMON}/OS/Linux/LinuxInput.cpp
        ${TF_COMMON}/OS/Linux/LinuxLog.c
        ${TF_COMMON}/OS/Linux/LinuxThread.c
        ${TF_COMMON}/OS/Linux/LinuxTime.c
        ${TF_COMMON}/OS/Linux/LinuxToolsFileSystem.c
        ${TF_COMMON}/OS/Linux/LinuxWindow.cpp
        # ThirdParty - Linux
        ${TF_COMMON}/OS/ThirdParty/OpenSource/cpu_features/src/impl_x86_linux_or_android.c
        ${TF_COMMON}/OS/ThirdParty/OpenSource/hidapi/linux/hid.c
    )
elseif(TF_PLATFORM STREQUAL "DARWIN")
    file(GLOB TF_OS_DARWIN_SOURCES
        "${TF_COMMON}/OS/Darwin/*.c"
        "${TF_COMMON}/OS/Darwin/*.cpp"
        "${TF_COMMON}/OS/Darwin/*.m"
        "${TF_COMMON}/OS/Darwin/*.mm"
    )
    list(APPEND TF_OS_SOURCES ${TF_OS_DARWIN_SOURCES})
endif()

# --- Utilities --------------------------------------------------------------
list(APPEND TF_OS_SOURCES
    ${TF_COMMON}/Utilities/FileSystem/FileSystem.c
    ${TF_COMMON}/Utilities/Log/Log.c
    ${TF_COMMON}/Utilities/Math/Algorithms.c
    ${TF_COMMON}/Utilities/Math/StbDs.c
    ${TF_COMMON}/Utilities/MemoryTracking/MemoryTracking.c
    ${TF_COMMON}/Utilities/Threading/ThreadSystem.c
    ${TF_COMMON}/Utilities/Timer.c
    # ThirdParty - Utilities
    ${TF_COMMON}/Utilities/ThirdParty/OpenSource/bstrlib/bstrlib.c
    ${TF_COMMON}/Utilities/ThirdParty/OpenSource/lz4/lz4.c
)
if(NOT WIN32)
    list(APPEND TF_OS_SOURCES
        ${TF_COMMON}/Utilities/FileSystem/UnixFileSystem.c
    )
endif()

# --- zstd (decompress only) ------------------------------------------------
file(GLOB TF_ZSTD_COMMON  "${TF_COMMON}/Utilities/ThirdParty/OpenSource/zstd/common/*.c")
file(GLOB TF_ZSTD_DECOMP  "${TF_COMMON}/Utilities/ThirdParty/OpenSource/zstd/decompress/*.c")
list(APPEND TF_OS_SOURCES ${TF_ZSTD_COMMON} ${TF_ZSTD_DECOMP})

# --- Animation system -------------------------------------------------------
list(APPEND TF_OS_SOURCES
    ${TF_COMMON}/Resources/AnimationSystem/Animation/AnimatedObject.cpp
    ${TF_COMMON}/Resources/AnimationSystem/Animation/Animation.cpp
    ${TF_COMMON}/Resources/AnimationSystem/Animation/Clip.cpp
    ${TF_COMMON}/Resources/AnimationSystem/Animation/ClipController.cpp
    ${TF_COMMON}/Resources/AnimationSystem/Animation/ClipMask.cpp
    ${TF_COMMON}/Resources/AnimationSystem/Animation/Rig.cpp
    ${TF_COMMON}/Resources/AnimationSystem/Animation/SkeletonBatcher.cpp
)

# --- OS target --------------------------------------------------------------
add_library(TheForge_OS STATIC ${TF_OS_SOURCES})
add_library(TheForge::OS ALIAS TheForge_OS)

target_include_directories(TheForge_OS
    PUBLIC
        ${TF_COMMON}
    PRIVATE
        ${TF_COMMON}/Application/ThirdParty/OpenSource
        ${TF_COMMON}/Game/ThirdParty/OpenSource/lua-5.3.5/src
        ${TF_COMMON}/OS/ThirdParty/OpenSource
        ${TF_COMMON}/Utilities/ThirdParty/OpenSource
        ${TF_COMMON}/Utilities/ThirdParty/OpenSource/zstd
)

target_compile_definitions(TheForge_OS
    PUBLIC
        _CRT_SECURE_NO_WARNINGS
        _CRT_NONSTDC_NO_DEPRECATE
    PRIVATE
        _HAS_EXCEPTIONS=0
)

if(MSVC)
    target_compile_options(TheForge_OS PRIVATE
        /W4
        /wd4201 /wd4324 /wd4127
        /GR-            # no RTTI
    )
endif()

if(TF_PLATFORM STREQUAL "WINDOWS")
    target_link_libraries(TheForge_OS
        PUBLIC
            xinput9_1_0
            ws2_32
            user32
            gdi32
            shell32
            ole32
            setupapi    # hidapi dependency
            hid         # hidapi dependency
    )
elseif(TF_PLATFORM STREQUAL "LINUX")
    target_link_libraries(TheForge_OS
        PUBLIC
            pthread
            dl
            X11
            Xcursor
    )
endif()

# ===========================================================================
#  TheForge_Renderer  (static library)
# ===========================================================================
set(TF_RENDERER_SOURCES
    ${TF_COMMON}/Graphics/GraphicsConfig.cpp
    ${TF_COMMON}/Renderer/ParticleSystem/ParticleSystem.cpp
    ${TF_COMMON}/Renderer/VisibilityBuffer/VisibilityBuffer.cpp
    ${TF_COMMON}/Resources/ResourceLoader/ResourceLoader.cpp
)

if(TF_BUILD_TOOLS)
    list(APPEND TF_RENDERER_SOURCES
        ${TF_COMMON}/Tools/Network/Network.c
        ${TF_COMMON}/Tools/ReloadServer/ReloadClient.cpp
    )
endif()

# --- Graphics backend -------------------------------------------------------
if(TF_PLATFORM STREQUAL "WINDOWS" AND NOT TF_USE_VULKAN)
    # Direct3D 12
    list(APPEND TF_RENDERER_SOURCES
        ${TF_COMMON}/Graphics/Direct3D12/Direct3D12.c
        ${TF_COMMON}/Graphics/Direct3D12/Direct3D12Hooks.c
        ${TF_COMMON}/Graphics/Direct3D12/Direct3D12Raytracing.c
        ${TF_COMMON}/Graphics/Direct3D12/Direct3D12_Cxx.cpp
    )
    set(TF_GFX_BACKEND "D3D12")
else()
    # Vulkan
    list(APPEND TF_RENDERER_SOURCES
        ${TF_COMMON}/Graphics/Vulkan/Vulkan.c
        ${TF_COMMON}/Graphics/Vulkan/VulkanRaytracing.c
        ${TF_COMMON}/Graphics/Vulkan/Vulkan_cxx.cpp
    )
    set(TF_GFX_BACKEND "VULKAN")
endif()

add_library(TheForge_Renderer STATIC ${TF_RENDERER_SOURCES})
add_library(TheForge::Renderer ALIAS TheForge_Renderer)

target_include_directories(TheForge_Renderer
    PUBLIC
        ${TF_COMMON}
    PRIVATE
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/ags
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/nvapi
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/renderdoc
)

target_compile_definitions(TheForge_Renderer
    PUBLIC
        _CRT_SECURE_NO_WARNINGS
    PRIVATE
        _HAS_EXCEPTIONS=0
)

if(MSVC)
    target_compile_options(TheForge_Renderer PRIVATE
        /W4
        /wd4201 /wd4324 /wd4127
        /GR-
    )
endif()

# Renderer depends on OS
target_link_libraries(TheForge_Renderer PUBLIC TheForge_OS)

# --- Graphics backend libraries --------------------------------------------
if(TF_GFX_BACKEND STREQUAL "D3D12")
    target_compile_definitions(TheForge_Renderer PUBLIC
        D3D12_AGILITY_SDK=1
        D3D12_AGILITY_SDK_VERSION=715
    )
    target_include_directories(TheForge_Renderer PRIVATE
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/Direct3d12Agility/include
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/DirectXShaderCompiler/inc
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/winpixeventruntime/Include
    )
    target_link_directories(TheForge_Renderer PUBLIC
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/DirectXShaderCompiler/lib/x64
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/ags/ags_lib/lib
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/nvapi/amd64
        ${TF_COMMON}/Graphics/ThirdParty/OpenSource/winpixeventruntime/bin
    )
    target_link_libraries(TheForge_Renderer
        PUBLIC
            d3d12
            dxgi
            dxguid
        PRIVATE
            dxcompiler
            WinPixEventRuntime
            amd_ags_x64
            nvapi64
    )
elseif(TF_GFX_BACKEND STREQUAL "VULKAN")
    target_compile_definitions(TheForge_Renderer PUBLIC
        FORGE_EXPLICIT_RENDERER_API_VULKAN
    )
    if(DEFINED ENV{VULKAN_SDK})
        target_include_directories(TheForge_Renderer PRIVATE $ENV{VULKAN_SDK}/Include)
        target_link_directories(TheForge_Renderer PUBLIC $ENV{VULKAN_SDK}/Lib)
    endif()
    target_link_libraries(TheForge_Renderer PRIVATE vulkan-1)
endif()

# ===========================================================================
#  Runtime DLLs  (convenience function for consumers)
# ===========================================================================
# Collect runtime DLLs that need to be copied next to the executable
set(TF_RUNTIME_DLLS "")
if(TF_GFX_BACKEND STREQUAL "D3D12")
    list(APPEND TF_RUNTIME_DLLS
        "${TF_COMMON}/Graphics/ThirdParty/OpenSource/ags/ags_lib/lib/amd_ags_x64.dll"
        "${TF_COMMON}/Graphics/ThirdParty/OpenSource/Direct3d12Agility/bin/x64/D3D12Core.dll"
        "${TF_COMMON}/Graphics/ThirdParty/OpenSource/Direct3d12Agility/bin/x64/d3d12SDKLayers.dll"
        "${TF_COMMON}/Graphics/ThirdParty/OpenSource/DirectXShaderCompiler/bin/x64/dxcompiler.dll"
        "${TF_COMMON}/Graphics/ThirdParty/OpenSource/winpixeventruntime/bin/WinPixEventRuntime.dll"
    )
endif()

# gpu.data for hardware detection
if(TF_PLATFORM STREQUAL "WINDOWS")
    list(APPEND TF_RUNTIME_DLLS
        "${TF_COMMON}/OS/Windows/pc_gpu.data"
    )
endif()

set(THEFORGE_RUNTIME_FILES ${TF_RUNTIME_DLLS} CACHE INTERNAL "The-Forge runtime files to copy")

# Helper function: copy runtime files to a target's output directory
function(theforge_copy_runtime_files TARGET_NAME)
    foreach(runtime_file IN LISTS THEFORGE_RUNTIME_FILES)
        if(EXISTS "${runtime_file}")
            get_filename_component(_fname "${runtime_file}" NAME)
            add_custom_command(
                TARGET ${TARGET_NAME} POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_if_different
                    "${runtime_file}"
                    "$<TARGET_FILE_DIR:${TARGET_NAME}>/${_fname}"
                COMMENT "Copying ${_fname}"
            )
        endif()
    endforeach()
endfunction()
