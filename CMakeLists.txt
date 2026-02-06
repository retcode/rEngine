cmake_minimum_required(VERSION 3.16)
project(GLQuake VERSION 1.09 LANGUAGES C)

# C standard (C99 for compatibility with legacy Quake code)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Option for SDL_mixer music support
option(USE_SDL_MIXER "Enable OGG/MP3 music playback via SDL_mixer" ON)

# SDL2 discovery
if(WIN32)
    # Windows: use vcpkg installation
    set(VCPKG_ROOT "C:/vcpkg/installed/x64-windows")
    set(SDL2_INCLUDE_DIR "${VCPKG_ROOT}/include/SDL2")
    set(SDL2_LIB_DIR "${VCPKG_ROOT}/lib")
    set(SDL2_BIN_DIR "${VCPKG_ROOT}/bin")
    set(SDL2_MANUAL_LINK_DIR "${VCPKG_ROOT}/lib/manual-link")

    if(USE_SDL_MIXER)
        set(SDL2_MIXER_INCLUDE_DIR "${VCPKG_ROOT}/include/SDL2")
        set(SDL2_MIXER_LIB_DIR "${VCPKG_ROOT}/lib")
        set(SDL2_MIXER_BIN_DIR "${VCPKG_ROOT}/bin")
    endif()
else()
    # macOS/Linux: use system packages via find_package or pkg-config
    find_package(PkgConfig QUIET)

    find_package(SDL2 QUIET)
    if(NOT SDL2_FOUND AND PKG_CONFIG_FOUND)
        pkg_check_modules(SDL2 REQUIRED sdl2)
    endif()
    if(NOT SDL2_FOUND)
        message(FATAL_ERROR "SDL2 not found. Install via: brew install sdl2 (macOS) or apt install libsdl2-dev (Linux)")
    endif()

    if(USE_SDL_MIXER)
        find_package(SDL2_mixer QUIET)
        if(NOT SDL2_mixer_FOUND AND PKG_CONFIG_FOUND)
            pkg_check_modules(SDL2_MIXER REQUIRED SDL2_mixer)
        endif()
        if(NOT SDL2_mixer_FOUND AND NOT SDL2_MIXER_FOUND)
            message(FATAL_ERROR "SDL2_mixer not found. Install via: brew install sdl2_mixer (macOS) or apt install libsdl2-mixer-dev (Linux)")
        endif()
    endif()
endif()

# Find OpenGL
find_package(OpenGL REQUIRED)

# 64-bit detection
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    add_definitions(-DQUAKE_64BIT=1)
    message(STATUS "Building for 64-bit")
else()
    add_definitions(-DQUAKE_64BIT=0)
    message(STATUS "Building for 32-bit")
endif()

# Compiler definitions
add_definitions(-DGLQUAKE -DUSE_SDL -Did386=0)

if(USE_SDL_MIXER)
    add_definitions(-DUSE_SDL_MIXER)
endif()

# Common engine sources
set(COMMON_SOURCES
    src/common/cmd.c
    src/common/common.c
    src/common/console.c
    src/common/crc.c
    src/common/cvar.c
    src/common/host.c
    src/common/host_cmd.c
    src/common/keys.c
    src/common/mathlib.c
    src/common/wad.c
    src/common/zone.c
)

# Client sources
set(CLIENT_SOURCES
    src/client/chase.c
    src/client/cl_demo.c
    src/client/cl_input.c
    src/client/cl_main.c
    src/client/cl_parse.c
    src/client/cl_tent.c
    src/client/menu.c
    src/client/sbar.c
    src/client/view.c
)

# Server sources
set(SERVER_SOURCES
    src/server/sv_main.c
    src/server/sv_move.c
    src/server/sv_phys.c
    src/server/sv_user.c
    src/server/world.c
)

# OpenGL renderer sources
set(GL_SOURCES
    src/renderer/gl_draw.c
    src/renderer/gl_mesh.c
    src/renderer/gl_model.c
    src/renderer/gl_refrag.c
    src/renderer/gl_rlight.c
    src/renderer/gl_rmain.c
    src/renderer/gl_rmisc.c
    src/renderer/gl_rsurf.c
    src/renderer/gl_screen.c
    src/renderer/gl_warp.c
    src/renderer/r_part.c
)

# Sound core sources
set(SOUND_SOURCES
    src/sound/snd_dma.c
    src/sound/snd_mem.c
    src/sound/snd_mix.c
)

# Networking sources (cross-platform UDP)
set(NET_SOURCES
    src/net/net_main.c
    src/net/net_loop.c
    src/net/net_dgrm.c
    src/net/net_vcr.c
    src/net/net_sdl.c
)

# Progs (QuakeC VM) sources
set(PROGS_SOURCES
    src/progs/pr_cmds.c
    src/progs/pr_edict.c
    src/progs/pr_exec.c
)

# SDL2 platform layer sources
set(SDL_SOURCES
    src/platform/sys_sdl.c
    src/platform/gl_vid_sdl.c
    src/platform/in_sdl.c
    src/platform/snd_sdl.c
    src/platform/cd_sdl.c
)

# Create the executable
add_executable(glquake
    ${COMMON_SOURCES}
    ${CLIENT_SOURCES}
    ${SERVER_SOURCES}
    ${GL_SOURCES}
    ${SOUND_SOURCES}
    ${NET_SOURCES}
    ${PROGS_SOURCES}
    ${SDL_SOURCES}
)

# Include directories
if(WIN32)
    target_include_directories(glquake PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${SDL2_INCLUDE_DIR}
        ${OPENGL_INCLUDE_DIR}
    )
    if(USE_SDL_MIXER)
        target_include_directories(glquake PRIVATE ${SDL2_MIXER_INCLUDE_DIR})
    endif()
else()
    target_include_directories(glquake PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${SDL2_INCLUDE_DIRS}
        ${OPENGL_INCLUDE_DIR}
    )
    if(USE_SDL_MIXER)
        if(TARGET SDL2_mixer::SDL2_mixer)
            # find_package succeeded with imported target — includes handled automatically
        elseif(SDL2_MIXER_INCLUDE_DIRS)
            target_include_directories(glquake PRIVATE ${SDL2_MIXER_INCLUDE_DIRS})
        endif()
    endif()
endif()

# Link libraries
if(WIN32)
    target_link_libraries(glquake PRIVATE
        ${SDL2_LIB_DIR}/SDL2.lib
        ${SDL2_MANUAL_LINK_DIR}/SDL2main.lib
        ${OPENGL_LIBRARIES}
    )
    if(USE_SDL_MIXER)
        target_link_libraries(glquake PRIVATE ${SDL2_MIXER_LIB_DIR}/SDL2_mixer.lib)
    endif()
else()
    target_link_libraries(glquake PRIVATE
        ${SDL2_LIBRARIES}
        ${OPENGL_LIBRARIES}
    )
    if(USE_SDL_MIXER)
        if(TARGET SDL2_mixer::SDL2_mixer)
            target_link_libraries(glquake PRIVATE SDL2_mixer::SDL2_mixer)
        elseif(SDL2_MIXER_LIBRARIES)
            target_link_libraries(glquake PRIVATE ${SDL2_MIXER_LIBRARIES})
        endif()
    endif()
endif()

# Platform-specific settings
if(WIN32)
    target_link_libraries(glquake PRIVATE ws2_32)
    target_compile_definitions(glquake PRIVATE _CRT_SECURE_NO_WARNINGS)
    # MSVC-specific: Disable optimizations and use strict floating-point
    if(MSVC)
        target_compile_options(glquake PRIVATE /Od /fp:strict)
    endif()
elseif(APPLE)
    target_link_libraries(glquake PRIVATE m)
    target_compile_definitions(glquake PRIVATE GL_SILENCE_DEPRECATION)
    # Legacy Quake code uses implicit function declarations throughout
    target_compile_options(glquake PRIVATE -Wno-implicit-function-declaration)
elseif(UNIX)
    target_link_libraries(glquake PRIVATE m)
    target_compile_options(glquake PRIVATE -Wno-implicit-function-declaration)
endif()

# Copy DLLs to output directory (Windows only)
if(WIN32)
    add_custom_command(TARGET glquake POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${SDL2_BIN_DIR}/SDL2.dll"
        $<TARGET_FILE_DIR:glquake>
    )

    if(USE_SDL_MIXER)
        add_custom_command(TARGET glquake POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${SDL2_MIXER_BIN_DIR}/SDL2_mixer.dll"
            $<TARGET_FILE_DIR:glquake>
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${SDL2_MIXER_BIN_DIR}/ogg.dll"
            $<TARGET_FILE_DIR:glquake>
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${SDL2_MIXER_BIN_DIR}/vorbis.dll"
            $<TARGET_FILE_DIR:glquake>
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${SDL2_MIXER_BIN_DIR}/vorbisfile.dll"
            $<TARGET_FILE_DIR:glquake>
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${SDL2_MIXER_BIN_DIR}/wavpackdll.dll"
            $<TARGET_FILE_DIR:glquake>
        )
    endif()
endif()

# Create id1 folder for game data
add_custom_command(TARGET glquake POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory
    "$<TARGET_FILE_DIR:glquake>/id1"
)

# Copy PAK files if they exist
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../original-quake-files/PAK0.PAK")
    add_custom_command(TARGET glquake POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_CURRENT_SOURCE_DIR}/../original-quake-files/PAK0.PAK"
        "$<TARGET_FILE_DIR:glquake>/id1/pak0.pak"
    )
endif()

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../original-quake-files/PAK1.PAK")
    add_custom_command(TARGET glquake POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_CURRENT_SOURCE_DIR}/../original-quake-files/PAK1.PAK"
        "$<TARGET_FILE_DIR:glquake>/id1/pak1.pak"
    )
endif()

# Create music folder for SDL_mixer
if(USE_SDL_MIXER)
    add_custom_command(TARGET glquake POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory
        "$<TARGET_FILE_DIR:glquake>/id1/music"
    )
endif()

# Print configuration summary
message(STATUS "")
message(STATUS "=== GLQuake Build Configuration ===")
message(STATUS "Version:        ${PROJECT_VERSION}")
message(STATUS "C Standard:     C${CMAKE_C_STANDARD}")
message(STATUS "Build Type:     ${CMAKE_BUILD_TYPE}")
if(WIN32)
    message(STATUS "SDL2 Path:      ${VCPKG_ROOT}")
else()
    message(STATUS "SDL2 Path:      (system)")
endif()
message(STATUS "SDL_mixer:      ${USE_SDL_MIXER}")
message(STATUS "")
