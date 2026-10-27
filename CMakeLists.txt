cmake_minimum_required(VERSION 3.22...3.25)
project(flood-tuber VERSION 1.1.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Find OBS if building standalone
if(NOT TARGET OBS::libobs)
    find_package(libobs QUIET)
    if(NOT libobs_FOUND)
        find_package(LibObs QUIET)
    endif()
    if(NOT libobs_FOUND AND NOT LibObs_FOUND)
        find_package(PkgConfig QUIET)
        if(PKG_CONFIG_FOUND)
            pkg_check_modules(LIBOBS QUIET libobs)
        endif()
    endif()

    if(TARGET OBS::libobs)
        # Already defined by find_package
    elseif(LIBOBS_FOUND)
        add_library(OBS::libobs INTERFACE IMPORTED)
        target_include_directories(OBS::libobs INTERFACE ${LIBOBS_INCLUDE_DIRS})
        target_link_libraries(OBS::libobs INTERFACE ${LIBOBS_LINK_LIBRARIES})
    elseif(DEFINED OBS_INCLUDE_DIR OR DEFINED LibObs_INCLUDE_DIRS)
        add_library(OBS::libobs INTERFACE IMPORTED)
        if(DEFINED OBS_INCLUDE_DIR)
            target_include_directories(OBS::libobs INTERFACE ${OBS_INCLUDE_DIR})
        endif()
        if(DEFINED LibObs_INCLUDE_DIRS)
            target_include_directories(OBS::libobs INTERFACE ${LibObs_INCLUDE_DIRS})
        endif()
    else()
        message(STATUS "libobs not found in system paths; fetching headers from obs-studio...")
        include(FetchContent)
        FetchContent_Declare(
            obs_headers
            GIT_REPOSITORY https://github.com/obsproject/obs-studio.git
            GIT_TAG        33.0.0
            GIT_SHALLOW    TRUE
        )
        FetchContent_GetProperties(obs_headers)
        if(NOT obs_headers_POPULATED)
            FetchContent_Populate(obs_headers)
        endif()
        FetchContent_Declare(
            simde
            GIT_REPOSITORY https://github.com/simd-everywhere/simde.git
            GIT_TAG        v0.8.2
            GIT_SHALLOW    TRUE
        )
        FetchContent_GetProperties(simde)
        if(NOT simde_POPULATED)
            FetchContent_Populate(simde)
        endif()

        if(NOT EXISTS "${obs_headers_BINARY_DIR}/obsconfig.h")
            file(WRITE "${obs_headers_BINARY_DIR}/obsconfig.h" "#pragma once\n")
        endif()

        add_library(OBS::libobs INTERFACE IMPORTED)
        target_include_directories(OBS::libobs INTERFACE 
            ${obs_headers_SOURCE_DIR}/libobs
            ${obs_headers_SOURCE_DIR}/libobs/util
            ${obs_headers_SOURCE_DIR}/libobs/graphics
            ${obs_headers_BINARY_DIR}
            ${simde_SOURCE_DIR}
        )
    endif()
endif()

if(NOT COMMAND set_target_properties_obs)
    function(set_target_properties_obs target)
        set_target_properties(${target} PROPERTIES PREFIX "")
    endfunction()
endif()

# 1. Plugin Definition
add_library(flood-tuber MODULE)
set_target_properties_obs(flood-tuber PROPERTIES FOLDER plugins PREFIX "")

# Version passed to C++ code via compile define (single source of truth)
target_compile_definitions(flood-tuber PRIVATE
    FLOOD_TUBER_VERSION="${PROJECT_VERSION}"
)

# 2. Dependencies (libwebp)
include(FetchContent)
FetchContent_Declare(
    libwebp
    GIT_REPOSITORY https://github.com/webmproject/libwebp.git
    GIT_TAG        v1.3.2
)
set(WEBP_BUILD_ANIM_UTILS OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_CWEBP OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_DWEBP OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_GIF2WEBP OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_IMG2WEBP OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_VWEBP OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_WEBPINFO OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_WEBPMUX OFF CACHE BOOL "" FORCE)
set(WEBP_BUILD_EXTRAS OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(libwebp)

# 3. Source Files
target_sources(flood-tuber PRIVATE
    flood-tuber.cpp
    flood-tuber.h
    flood-tuber-props.cpp
    flood-tuber-props.h
    webp-decoder.cpp
    webp-decoder.h
    lodepng.cpp
    lodepng.h
    apng-decoder.cpp
    apng-decoder.h
)

# 4. Libraries (Link OBS::libobs)
target_link_libraries(flood-tuber PRIVATE
    OBS::libobs
    webp
    webpdemux
)

# 5. Data directory copy (Build time)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/data")
    file(GLOB_RECURSE data_files "${CMAKE_CURRENT_SOURCE_DIR}/data/*")
    foreach(data_file ${data_files})
        file(RELATIVE_PATH dest_file "${CMAKE_CURRENT_SOURCE_DIR}/data" "${data_file}")
        get_filename_component(dest_dir "${dest_file}" DIRECTORY)
        add_custom_command(
            TARGET flood-tuber POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E make_directory "${CMAKE_BINARY_DIR}/data/obs-plugins/flood-tuber/${dest_dir}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${data_file}" "${CMAKE_BINARY_DIR}/data/obs-plugins/flood-tuber/${dest_file}"
            VERBATIM
        )
    endforeach()
endif()

# 6. Install rules (Linux / Packaging)
include(GNUInstallDirs)
install(TARGETS flood-tuber
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}/obs-plugins"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}/obs-plugins"
)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/data")
    install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/data/"
        DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/obs/obs-plugins/flood-tuber"
    )
endif()