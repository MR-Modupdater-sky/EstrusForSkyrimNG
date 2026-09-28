# This file configures the "build" of this SKSE plugin
# - how to compile the C++ plugin (in addition to settings found in CMakePresets.json)
# - where to output the resulting .dll file

# It's recommended to set a minimum CMake version.
# If you use CMake features from higher versions, update this to match.
cmake_minimum_required(VERSION 3.21)

# [BEGIN set_project_name]
# Set your project name. This will be the name of your SKSE .dll file.
project(EstrusForSkyrimNG VERSION 0.0.1 LANGUAGES CXX)

# [END set_project_name]

# #
# YOU DO NOT NEED TO EDIT ANYTHING BELOW HERE
# #

# If you're not using a mod manager, you probably want the SKSE plugin to go
# inside of your Skyrim "Data" folder.
#
# To do this automatically, set the `SKYRIM_FOLDER` environment variable
# to the path of your Skyrim Special Edition folder
if(DEFINED ENV{SKYRIM_FOLDER} AND IS_DIRECTORY "$ENV{SKYRIM_FOLDER}/Data")
    set(OUTPUT_FOLDER "$ENV{SKYRIM_FOLDER}/Data")
endif()

# If you're using Mod Organizer 2 or Vortex, you might want this to go inside
# of your "mods" folder, inside of a subfolder named "<your mod>".
#
# To do this automatically, set the `SKYRIM_MODS_FOLDER` environment variable
# to the path of your "mods" folder
if(DEFINED ENV{SKYRIM_MODS_FOLDER} AND IS_DIRECTORY "$ENV{SKYRIM_MODS_FOLDER}")
    set(OUTPUT_FOLDER "$ENV{SKYRIM_MODS_FOLDER}/${PROJECT_NAME}")
endif()

# Otherwise, you can set OUTPUT_FOLDER to any place you'd like :)
# set(OUTPUT_FOLDER "C:/path/to/any/folder")

# Papertongues666: Use the extracted D: source tree locally; clean builds fetch
# pinned v10.0.0. The vcpkg manifests list this version's direct build dependencies.
include(FetchContent)
# Papertongues666: CommonLib's unit tests are not part of this plugin build.
set(BUILD_TESTS OFF CACHE BOOL "Disable CommonLibSSE-NG tests in this plugin build" FORCE)
# Papertongues666: This plugin targets Skyrim SE/AE, not VR, so do not link OpenVR.
set(ENABLE_SKYRIM_VR OFF CACHE BOOL "This plugin does not target Skyrim VR" FORCE)
if(DEFINED COMMONLIB_SOURCE_DIR AND IS_DIRECTORY "${COMMONLIB_SOURCE_DIR}")
    set(_commonlib_source_dir "${COMMONLIB_SOURCE_DIR}")
    add_subdirectory("${COMMONLIB_SOURCE_DIR}" "${CMAKE_BINARY_DIR}/commonlib")
else()
    FetchContent_Declare(
        commonlibsse_ng
        GIT_REPOSITORY https://github.com/alandtse/CommonLibSSE-NG.git
        GIT_TAG v10.0.0
        GIT_SHALLOW TRUE
    )
    FetchContent_MakeAvailable(commonlibsse_ng)
    set(_commonlib_source_dir "${commonlibsse_ng_SOURCE_DIR}")
endif()

# Papertongues666: The source-build path does not register this plugin helper itself.
if(NOT COMMAND add_commonlibsse_plugin)
    include("${_commonlib_source_dir}/cmake/CommonLibSSE.cmake")
endif()

# Setup your SKSE plugin as an SKSE plugin!
add_commonlibsse_plugin(${PROJECT_NAME}
    SOURCES 
    src/main.cpp
    src/PCH.h
    src/Utility.cpp
    src/Utility.h
    src/CommonFollowerUtil.cpp
    src/CommonFollowerUtil.h
    src/Papyrus.cpp
    src/Papyrus.h
) # <--- specifies plugin.cpp
target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_23) # <--- use C++23 standard
target_precompile_headers(${PROJECT_NAME} PRIVATE src/PCH.h) # <--- PCH.h is required!

# When your SKSE .dll is compiled, this will automatically copy the .dll into your mods folder.
# Only works if you configure DEPLOY_ROOT above (or set the SKYRIM_MODS_FOLDER environment variable)
if(DEFINED OUTPUT_FOLDER)
    # If you specify an <OUTPUT_FOLDER> (including via environment variables)
    # then we'll copy your mod files into Skyrim or a mod manager for you!

    # Copy the SKSE plugin .dll files into the SKSE/Plugins/ folder
    set(DLL_FOLDER "${OUTPUT_FOLDER}/SKSE/Plugins")

    message(STATUS "SKSE plugin output folder: ${DLL_FOLDER}")

    add_custom_command(
        TARGET "${PROJECT_NAME}"
        POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E make_directory "${DLL_FOLDER}"
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}/$<TARGET_FILE_NAME:${PROJECT_NAME}>"
        VERBATIM
    )

    # If you perform a "Debug" build, also copy .pdb file (for debug symbols)
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        add_custom_command(
            TARGET "${PROJECT_NAME}"
            POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_PDB_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}/$<TARGET_PDB_FILE_NAME:${PROJECT_NAME}>"
            VERBATIM
        )
    endif()
endif()
