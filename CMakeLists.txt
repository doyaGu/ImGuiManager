cmake_minimum_required(VERSION 3.12)

project(ImGuiManager VERSION 0.2.0)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED NO)
set(CMAKE_CXX_EXTENSIONS YES)

# Add path for custom modules
list(INSERT CMAKE_MODULE_PATH 0
        "${CMAKE_CURRENT_SOURCE_DIR}/CMake"
        "${CMAKE_CURRENT_SOURCE_DIR}/CMake/Modules"
        )

include(GenerateExportHeader)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Use folders to organize targets in an IDE
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "CMakeTargets")

if (NOT WIN32)
    message(FATAL_ERROR "Only support Windows.")
endif ()

# Use relative paths
if (WIN32)
    set(CMAKE_USE_RELATIVE_PATHS TRUE)
    set(CMAKE_SUPPRESS_REGENERATION TRUE)
endif ()

if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to 'Release' as no build type was specified")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the build type (Debug/Release)" FORCE)
endif ()

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "${PROJECT_SOURCE_DIR}/install" CACHE PATH "The root directory of the installation" FORCE)
    message(STATUS "Setting default install directory to ${CMAKE_INSTALL_PREFIX} as no install directory was specified")
endif ()

# Generate a CompilationDatabase (compile_commands.json)
set(CMAKE_EXPORT_COMPILE_COMMANDS TRUE)

set(VIRTOOLS_SDK_PATH "${VIRTOOLS_SDK_PATH}" CACHE PATH "Path to the Virtools SDK")
set(VIRTOOLS_SDK_FETCH_FROM_GIT "${VIRTOOLS_SDK_FETCH_FROM_GIT}" CACHE BOOL "Set to ON to fetch copy of SDK from git if not otherwise locatable")
set(VIRTOOLS_SDK_FETCH_FROM_GIT_PATH "${VIRTOOLS_SDK_FETCH_FROM_GIT_PATH}" CACHE FILEPATH "location to download SDK")

if (NOT VIRTOOLS_SDK_PATH)
    if (NOT VIRTOOLS_SDK_FETCH_FROM_GIT)
        message(FATAL_ERROR "Virtools SDK location was not specified. Please set VIRTOOLS_SDK_PATH or set VIRTOOLS_SDK_FETCH_FROM_GIT to on to fetch from git.")
    else ()
        include(FetchContent)
        set(FETCHCONTENT_BASE_DIR_SAVE ${FETCHCONTENT_BASE_DIR})
        if (VIRTOOLS_SDK_FETCH_FROM_GIT_PATH)
            get_filename_component(FETCHCONTENT_BASE_DIR "${VIRTOOLS_SDK_FETCH_FROM_GIT_PATH}" REALPATH BASE_DIR "${CMAKE_SOURCE_DIR}")
        endif ()
        FetchContent_Declare(
                Virtools_SDK
                GIT_REPOSITORY https://github.com/doyaGu/Virtools-SDK-2.1.git
                GIT_TAG main
        )
        if (NOT Virtools_SDK)
            message("Downloading Virtools SDK")
            FetchContent_Populate(Virtools_SDK)
            set(VIRTOOLS_SDK_PATH "${virtools_sdk_SOURCE_DIR}" CACHE PATH "Path to the Virtools SDK" FORCE)
        endif ()
        set(FETCHCONTENT_BASE_DIR ${FETCHCONTENT_BASE_DIR_SAVE})
    endif ()
endif ()

find_package(VirtoolsSDK REQUIRED HINTS ${VIRTOOLS_SDK_PATH})

# MinHook
add_subdirectory(minhook EXCLUDE_FROM_ALL)
set_target_properties(minhook PROPERTIES FOLDER "Dependencies")

add_compile_definitions(
        $<$<C_COMPILER_ID:MSVC>:_CRT_SECURE_NO_WARNINGS>
        $<$<C_COMPILER_ID:MSVC>:_CRT_NONSTDC_NO_WARNINGS>
        $<$<C_COMPILER_ID:MSVC>:_CRT_SECURE_NO_DEPRECATE>
        $<$<C_COMPILER_ID:MSVC>:_CRT_NONSTDC_NO_DEPRECATE>
		)

configure_file(ImGuiManager.rc.in "${CMAKE_CURRENT_LIST_DIR}/ImGuiManager.rc")

set(IMGUI_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/imgui)

set(IMGUI_HDRS
        ${IMGUI_SOURCE_DIR}/imgui.h
        ${IMGUI_SOURCE_DIR}/imgui_internal.h
        ${IMGUI_SOURCE_DIR}/imstb_rectpack.h
        ${IMGUI_SOURCE_DIR}/imstb_textedit.h
        ${IMGUI_SOURCE_DIR}/imstb_truetype.h
        ${IMGUI_SOURCE_DIR}/backends/imgui_impl_win32.h
        ${CMAKE_CURRENT_SOURCE_DIR}/imgui_impl_ck2.h
        )

set(IMGUI_SRCS
        ${IMGUI_SOURCE_DIR}/imgui.cpp
        ${IMGUI_SOURCE_DIR}/imgui_draw.cpp
        ${IMGUI_SOURCE_DIR}/imgui_widgets.cpp
        ${IMGUI_SOURCE_DIR}/imgui_demo.cpp
        ${IMGUI_SOURCE_DIR}/imgui_tables.cpp
        ${IMGUI_SOURCE_DIR}/backends/imgui_impl_win32.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/imgui_impl_ck2.cpp
        )

set(IMGUIMANAGER_HDRS
        ${CMAKE_CURRENT_SOURCE_DIR}/ImGuiManager.h
        ${CMAKE_CURRENT_SOURCE_DIR}/ImGuiOverlay.h
        )

set(IMGUIMANAGER_SRCS
        ${CMAKE_CURRENT_SOURCE_DIR}/Plugin.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/ImGuiManager.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/ImGuiWin32Hook.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/ImGuiFunctionTable.cpp
		${CMAKE_CURRENT_SOURCE_DIR}/ImGuiManager.rc
        )

add_library(ImGuiManager SHARED)
target_sources(ImGuiManager PRIVATE ${IMGUIMANAGER_SRCS} ${IMGUIMANAGER_HDRS} ${IMGUI_SRCS} ${IMGUI_HDRS})
target_include_directories(ImGuiManager PUBLIC
        $<BUILD_INTERFACE:${IMGUI_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
        )
target_link_libraries(ImGuiManager PRIVATE CK2 VxMath minhook)

if (NOT SKIP_INSTALL_ALL)
	install(TARGETS ImGuiManager
			RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/Managers
			)
		
	if (NOT SKIP_INSTALL_HEADERS)
		install(FILES
				${IMGUI_SOURCE_DIR}/imgui.h
				${CMAKE_CURRENT_SOURCE_DIR}/ImGuiManager.h
				${CMAKE_CURRENT_SOURCE_DIR}/ImGuiOverlay.h
				DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
	endif ()	
endif ()
	
