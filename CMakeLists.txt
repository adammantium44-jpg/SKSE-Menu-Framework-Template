# =========================
# CommonLibSSE location
# =========================
# Pakai submodule di extern/CommonLibSSE
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/extern/CommonLibSSE/CMakeLists.txt")
    set(COMMONLIB_SSE_FOLDER "${CMAKE_CURRENT_SOURCE_DIR}/extern/CommonLibSSE")
else()
    message(FATAL_ERROR "Missing extern/CommonLibSSE (CommonLibSSE-NG submodule)")
endif()

# SKYRIM_MODS_FOLDER masih perlu (buat copy output + SKSE Menu Framework)
if(NOT DEFINED ENV{SKYRIM_MODS_FOLDER})
  message(FATAL_ERROR "Missing SKYRIM_MODS_FOLDER environment variable")
endif()

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

set(wildlander_output false)
set(steam_owrt_output false)
set(steam_mods_output true)

# =========================
# Project config
# =========================
cmake_minimum_required(VERSION 3.21)

set(AUTHOR_NAME "AuthorName")
set(PRODUCT_NAME "SKSEMenuFrameworkTemplate0.2")
set(BEAUTIFUL_NAME "SKSE Menu Framework Template")

project(${PRODUCT_NAME} VERSION 0.2.0.0 LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.rc.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.rc
    @ONLY
)

set(RESOURCE_FILE ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
set(RESOURCE_OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/version.res)

add_custom_command(
    OUTPUT ${RESOURCE_OUTPUT}
    COMMAND rc /fo ${RESOURCE_OUTPUT} ${RESOURCE_FILE}
    DEPENDS ${RESOURCE_FILE}
)

add_custom_target(Resource ALL DEPENDS ${RESOURCE_OUTPUT})

# =========================
# CommonLibSSE + CLIB-UTIL
# =========================
include(FetchContent)

set(ENABLE_SKYRIM_SE ON CACHE BOOL " " FORCE)
set(ENABLE_SKYRIM_AE ON CACHE BOOL " " FORCE)
set(ENABLE_SKYRIM_VR ON CACHE BOOL " " FORCE)
set(BUILD_TESTS OFF CACHE BOOL " " FORCE)

# RapidCSV include dir (lokal, dari repo sendiri)
if(NOT DEFINED RAPIDCSV_INCLUDE_DIRS)
    set(RAPIDCSV_INCLUDE_DIRS
        "${CMAKE_CURRENT_SOURCE_DIR}/extern/rapidcsv"
    )
endif()

# CommonLibSSE from submodule
add_subdirectory(${COMMONLIB_SSE_FOLDER} ${CMAKE_CURRENT_BINARY_DIR}/_deps/clib-build)
get_target_property(COMMONLIB_SRC_DIR CommonLibSSE SOURCE_DIR)

# CLIB-UTIL include path (pakai yang datang dari CommonLib)
get_target_property(CLIB_UTIL_INCLUDE_DIRS CommonLibSSE INTERFACE_INCLUDE_DIRECTORIES)

file(
  GLOB_RECURSE
  sources
  ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/version.rc
)

source_group(
  TREE ${CMAKE_CURRENT_SOURCE_DIR}
  FILES ${headers} ${sources}
)

set(BUILD_TESTS OFF)
include(${COMMONLIB_SRC_DIR}/cmake/CommonLibSSE.cmake)

add_commonlibsse_plugin(
  ${PROJECT_NAME}
  AUTHOR AUTHOR_NAME
  SOURCES ${headers} ${sources}
)

target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_23)
target_precompile_headers(${PROJECT_NAME} PRIVATE include/PCH.h)

# =========================
# Includes & ImGui (lokal)
# =========================
target_include_directories(
    ${PROJECT_NAME}
    PRIVATE
    ${CLIB_UTIL_INCLUDE_DIRS}
    ${CMAKE_CURRENT_BINARY_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/extern/imgui
)

# Dear ImGui sources as a separate static library
add_library(imgui-lib
    extern/imgui/imgui.cpp
    extern/imgui/imgui_draw.cpp
    extern/imgui/imgui_tables.cpp
    extern/imgui/imgui_widgets.cpp
    extern/imgui/imgui_demo.cpp
)

target_include_directories(imgui-lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/extern/imgui)

target_link_libraries(${PROJECT_NAME} PRIVATE imgui-lib)

include(cmake/headerlist.cmake)
include(cmake/sourcelist.cmake)

install(TARGETS ${PROJECT_NAME}
        DESTINATION "${CMAKE_INSTALL_LIBDIR}")

# =========================
# SKSE Menu Framework
# =========================
function(addMenuFramework MENU_FRAMEWORK_FOLDER)

    # Di CI, kita tidak pakai ImGui dari vcpkg/IMGUI_LIB_FOLDER lagi,
    # jadi bagian find_package(ImGui) dihilangkan.

    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        set(PROJECT_SUFFIX " DEV")
    else()
        set(PROJECT_SUFFIX "")
    endif()

    add_library(SKSEMenuFramework SHARED IMPORTED GLOBAL)

    set_target_properties(SKSEMenuFramework PROPERTIES
        IMPORTED_LOCATION "${MENU_FRAMEWORK_FOLDER}/SKSE Menu Framework${PROJECT_SUFFIX}/SKSE/Plugins/SKSEMenuFramework.dll"
        IMPORTED_IMPLIB "${MENU_FRAMEWORK_FOLDER}/SKSE Menu Framework${PROJECT_SUFFIX}/SKSE/Plugins/SKSEMenuFramework.lib"
    )

    target_include_directories(
	    ${PROJECT_NAME}
	    PRIVATE
        "${MENU_FRAMEWORK_FOLDER}/SKSE Menu Framework${PROJECT_SUFFIX}/SKSE/Plugins"
    )

    target_link_libraries(${PROJECT_NAME} PRIVATE SKSEMenuFramework)

endfunction()

addMenuFramework("$ENV{SKYRIM_MODS_FOLDER}")

# =========================
# Output copy helpers
# =========================
function(copyOutputs TARGET_FOLDER)
    set(DLL_FOLDER "${TARGET_FOLDER}/SKSE/Plugins")

    message(STATUS "SKSE plugin output folder: ${DLL_FOLDER}")

    add_custom_command(
        TARGET "${PROJECT_NAME}"
        POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E make_directory "${DLL_FOLDER}"
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}/$<TARGET_FILE_NAME:${PROJECT_NAME}>"
        VERBATIM
    )

    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        add_custom_command(
            TARGET "${PROJECT_NAME}"
            POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_PDB_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}/$<TARGET_PDB_FILE_NAME:${PROJECT_NAME}>"
            VERBATIM
        )
    endif()
endfunction()

if(DEFINED ENV{SKYRIM_FOLDER} AND IS_DIRECTORY "$ENV{SKYRIM_FOLDER}/Data")
    copyOutputs("$ENV{SKYRIM_FOLDER}/Data")
endif()
if(DEFINED ENV{WILDLANDER_OWRT_FOLDER} AND IS_DIRECTORY "$ENV{WILDLANDER_OWRT_FOLDER}" AND wildlander_output)
    copyOutputs("$ENV{WILDLANDER_OWRT_FOLDER}/${BEAUTIFUL_NAME}")
endif()
if(DEFINED ENV{SKYRIM_OWRT_FOLDER} AND IS_DIRECTORY "$ENV{SKYRIM_OWRT_FOLDER}" AND steam_owrt_output)
    copyOutputs("$ENV{SKYRIM_OWRT_FOLDER}/${BEAUTIFUL_NAME}")
endif()
if(DEFINED ENV{SKYRIM_MODS_FOLDER} AND IS_DIRECTORY "$ENV{SKYRIM_MODS_FOLDER}" AND steam_mods_output)
    copyOutputs("$ENV{SKYRIM_MODS_FOLDER}/${BEAUTIFUL_NAME}")
endif()
if(DEFINED ENV{SKYRIM_MODS_FOLDER2} AND IS_DIRECTORY "$ENV{SKYRIM_MODS_FOLDER2}" AND steam_mods_output)
    copyOutputs("$ENV{SKYRIM_MODS_FOLDER2}/${BEAUTIFUL_NAME}")
endif()
