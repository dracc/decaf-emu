cmake_minimum_required(VERSION 3.2)

project(decaf-emu C CXX)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules")
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake OPTIONAL RESULT_VARIABLE USING_CONAN)
if(USING_CONAN)
    conan_basic_setup(NO_OUTPUT_DIRS)
endif()

find_package(CURL REQUIRED)
find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED) # OpenSSL::SSL
find_package(ZLIB REQUIRED) # ZLIB::ZLIB

option(DECAF_BUILD_TESTS "Build tests" OFF)
option(DECAF_BUILD_TOOLS "Build tools" OFF)
option(DECAF_BUILD_WUT_TESTS "Build tests which rely on wut" OFF)
option(DECAF_JIT_ALLOW_PROFILING "Build with JIT profiling support" OFF)
option(DECAF_FFMPEG "Build with ffmpeg support" ON)
option(DECAF_GL "Build with OpenGL rendering support" ON)
option(DECAF_GLAD_DEBUG "Build with GLAD_DEBUG support" ON)
option(DECAF_SDL "Build with SDL support" ON)
option(DECAF_VALGRIND "Build with Valgrind support" OFF)
option(DECAF_VULKAN "Build with Vulkan rendering support" ON)
option(DECAF_QT "Build with Qt support" OFF)
option(DECAF_ENABLE_PCH "Build with precompild headers" ON)
option(DECAF_ENABLE_LTCG "Build with LTCG on Windows" ON)

set(DEVKITPPC $ENV{DEVKITPPC} CACHE STRING "Path to devkitPPC install")
set(WUT_ROOT $ENV{WUT_ROOT} CACHE STRING "Path to wut install")

if(DECAF_BUILD_WUT_TESTS AND (NOT DEVKITPPC OR NOT WUT_ROOT))
    message(FATAL_ERROR "You must have defined DEVKITPPC and WUT_ROOT to build wut tests.")
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/obj)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/obj)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/obj)

# Setup install directories
include(GNUInstallDirs)
if(WIN32)
    set(DECAF_INSTALL_BINDIR "${CMAKE_INSTALL_PREFIX}")
    set(DECAF_INSTALL_DOCSDIR "${CMAKE_INSTALL_PREFIX}")
    set(DECAF_INSTALL_RESOURCESDIR "${CMAKE_INSTALL_PREFIX}/resources")
else()
    set(DECAF_INSTALL_BINDIR "${CMAKE_INSTALL_BINDIR}")
    set(DECAF_INSTALL_DOCSDIR "${CMAKE_INSTALL_DATAROOTDIR}/doc/${PROJECT_NAME}")
    set(DECAF_INSTALL_RESOURCESDIR "${CMAKE_INSTALL_DATAROOTDIR}/${PROJECT_NAME}/resources")
endif()

if(DECAF_JIT_ALLOW_PROFILING)
    add_definitions(-DDECAF_JIT_ALLOW_PROFILING)
endif()

if(DECAF_FFMPEG)
    find_package(FFMPEG REQUIRED)
    add_definitions(-DDECAF_FFMPEG)
endif()

if(DECAF_GL)
    if(APPLE)
        message(FATAL_ERROR "The OpenGL backend is not supported on macOS. Please use -DDECAF_GL=OFF to suppress this error.")
    endif()

    find_package(OpenGL REQUIRED)
    add_definitions(-DDECAF_GL)
endif()

if(DECAF_SDL OR DECAF_QT)
    # decaf-qt still uses SDL for input
    find_package(SDL2 REQUIRED)
endif()

if(DECAF_SDL)
    add_definitions(-DDECAF_SDL)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    find_package(XCB QUIET)
    find_package(X11 QUIET)
    find_package(WAYLAND QUIET)

    if(${XCB_FOUND})
        add_definitions(-DDECAF_PLATFORM_XCB)
        set(DECAF_PLATFORM_XCB TRUE)
    endif()

    if(${X11_FOUND})
        add_definitions(-DDECAF_PLATFORM_XLIB)
        set(DECAF_PLATFORM_XLIB TRUE)
    endif()

    if(${WAYLAND_FOUND})
        add_definitions(-DDECAF_PLATFORM_WAYLAND)
        set(DECAF_PLATFORM_WAYLAND TRUE)
    endif()
endif()

if(DECAF_VULKAN)
    find_package(Vulkan 1.1.92.1 REQUIRED) # Vulkan_INCLUDE_DIRS and Vulkan_LIBRARIES
    add_library(vulkan INTERFACE IMPORTED)
    set_target_properties(vulkan PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES ${Vulkan_INCLUDE_DIRS}
        INTERFACE_LINK_LIBRARIES ${Vulkan_LIBRARIES})

    if(MSVC)
        add_definitions(-DVK_USE_PLATFORM_WIN32_KHR)
    endif()

    if(DECAF_PLATFORM_XCB)
        add_definitions(-DVK_USE_PLATFORM_XCB_KHR)
    endif()

    if(DECAF_PLATFORM_XLIB)
        add_definitions(-DVK_USE_PLATFORM_XLIB_KHR)
    endif()

    if(DECAF_PLATFORM_WAYLAND)
        add_definitions(-DVK_USE_PLATFORM_WAYLAND_KHR)
    endif()

    add_definitions(-DDECAF_VULKAN)
endif()

if(DECAF_VALGRIND)
    add_definitions(-DDECAF_VALGRIND)
endif()

if(DECAF_QT)
    find_package(Qt5Widgets REQUIRED)
    find_package(Qt5Svg REQUIRED)
    add_definitions(-DDECAF_QT)
endif()

# Build third party libraries
add_subdirectory("libraries")

# Setup compile options
if(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++latest")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /FS") # Parallel source builds
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /GA") # Optimises TLS access

    if(DECAF_ENABLE_LTCG)
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /GL")
        set(CMAKE_STATIC_LINKER_FLAGS_RELEASE "${CMAKE_STATIC_LINKER_FLAGS_RELEASE} /LTCG")
        set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /LTCG")
        set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /LTCG")
    endif()

    add_definitions(-DNOMINMAX)
    add_definitions(-DUNICODE -D_UNICODE)

    # Disable warnings about using deprecated std::wstring_convert
    add_definitions(-D_SILENCE_CXX17_CODECVT_HEADER_DEPRECATION_WARNING)
else()
    add_definitions(-DDECAF_USE_STDLAYOUT_BITFIELD)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")

    if(APPLE)
        add_definitions(-D_DARWIN_C_SOURCE)
    else()
        link_libraries(stdc++fs)
    endif()
endif()

# Macro to map filters to folder structure for MSVC projects
macro(GroupSources curdir)
    if(MSVC)
        file(GLOB children RELATIVE ${PROJECT_SOURCE_DIR}/${curdir} ${PROJECT_SOURCE_DIR}/${curdir}/*)

        foreach(child ${children})
            if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${curdir}/${child})
                GroupSources(${curdir}/${child})
            else()
                string(REPLACE "/" "\\" groupname ${curdir})
                source_group(${groupname} FILES ${PROJECT_SOURCE_DIR}/${curdir}/${child})
            endif()
        endforeach()
    endif()
endmacro()

# Generate build information
include(GetGitRevisionDescription)

function(get_timestamp _var)
    string(TIMESTAMP timestamp UTC)
    set(${_var} "${timestamp}" PARENT_SCOPE)
endfunction()

get_git_head_revision(GIT_REF_SPEC GIT_REV)
git_describe(GIT_DESC --always --long --dirty)
git_branch_name(GIT_BRANCH)
get_timestamp(BUILD_DATE)

set(BUILD_VERSION "0")
if ($ENV{CI})
   if ($ENV{TRAVIS})
      set(BUILD_TAG $ENV{TRAVIS_TAG})
   elseif($ENV{APPVEYOR})
      set(BUILD_TAG $ENV{APPVEYOR_REPO_TAG_NAME})
   endif()

   if (BUILD_TAG)
      string(REGEX MATCH "${CMAKE_MATCH_1}-([0-9]+)" OUTVAR ${BUILD_TAG})
      if (${CMAKE_MATCH_COUNT} GREATER 0)
          set(BUILD_VERSION ${CMAKE_MATCH_1})
      endif()
   endif()
endif()

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/src/decaf_buildinfo.h.in"
               "${CMAKE_CURRENT_BINARY_DIR}/generated/decaf_buildinfo.h" @ONLY)
include_directories("${CMAKE_CURRENT_BINARY_DIR}/generated")

# cotire for build optimisation
include(cotire)
macro(enable_pch project_name)
    if(DECAF_ENABLE_PCH)
        set(IGNORED_PATHS "")
        set(INCLUDED_PATHS "")

        list(APPEND IGNORED_PATHS  "${CMAKE_SOURCE_DIR}")
        list(APPEND IGNORED_PATHS  "${AVCODEC_INCLUDE_DIR}")
        list(APPEND IGNORED_PATHS  "${AVFILTER_INCLUDE_DIR}")
        list(APPEND IGNORED_PATHS  "${AVUTIL_INCLUDE_DIR}")
        list(APPEND IGNORED_PATHS  "${SWSCALE_INCLUDE_DIR}")
        list(APPEND INCLUDED_PATHS "${CMAKE_SOURCE_DIR}/libraries")
        list(APPEND INCLUDED_PATHS "${CMAKE_SOURCE_DIR}/src/common")

        set_target_properties(${project_name} PROPERTIES
            COTIRE_ADD_UNITY_BUILD FALSE
            COTIRE_PREFIX_HEADER_IGNORE_PATH "${IGNORED_PATHS}"
            COTIRE_PREFIX_HEADER_INCLUDE_PATH "${INCLUDED_PATHS}")
        cotire(${project_name})
    endif()
endmacro()

add_subdirectory("src")
add_subdirectory("resources")

if(DECAF_BUILD_TOOLS)
    add_subdirectory("tools")
endif()

if(DECAF_BUILD_TESTS OR DECAF_BUILD_WUT_TESTS)
    enable_testing()
    add_subdirectory("tests")
endif()
