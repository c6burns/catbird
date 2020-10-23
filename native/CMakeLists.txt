cmake_minimum_required(VERSION 3.1)
project(catbird LANGUAGES C CXX)

set(CB_SRCDIR ${CMAKE_CURRENT_SOURCE_DIR})
set(CB_BINDIR ${CMAKE_CURRENT_BINARY_DIR})
set(CB_DEPDIR ${CB_SRCDIR}/../deps)
set(CB_DEPDIR_BIN ${CB_BINDIR}/deps)
set(CB_GIT_PROTO "https" CACHE STRING "Determines which transport git will use to pull dependencies")

option(CB_BUILD_DEPS "Builds runtime dependencies as part of this project's build" ON)
option(CB_BUILD_DEPS_AWS "Builds aws-c-* runtime dependencies as part of this project's build" ON)
option(CB_BUILD_DEPS_WX "Builds wxWidgets runtime dependencies as part of this project's build" ON)
if (NOT CB_BUILD_DEPS)
    set(CB_BUILD_DEPS_AWS OFF)
    set(CB_BUILD_DEPS_WX OFF)
endif()

# private: add_project_dependency_impl
# 0: repo name (not URL)
macro(cb_add_git_dependency)
    set(REPO_URL https://github.com/c6burns/${ARGV0})
    if (CB_GIT_PROTO EQUAL ssh)
        set(REPO_URL ssh://git@github.com/c6burns/${ARGV0})
    endif()

    if (NOT EXISTS ${CB_DEPDIR}/${ARGV0}/CMakeLists.txt)
        execute_process(COMMAND git clone --progress --recursive ${REPO_URL} WORKING_DIRECTORY ${CB_DEPDIR})
    endif()
    add_subdirectory(${CB_DEPDIR}/${ARGV0} ${CB_DEPDIR_BIN}/${ARGV0})
endmacro()


if (POLICY CMP0069)
    cmake_policy(SET CMP0069 NEW) # Enable LTO/IPO if available in the compiler, see AwsCFlags
endif()

if (POLICY CMP0077)
    cmake_policy(SET CMP0077 NEW) # Enable options to get their values from normal variables
endif()

if (DEFINED CMAKE_PREFIX_PATH)
    file(TO_CMAKE_PATH ${CMAKE_PREFIX_PATH} CMAKE_PREFIX_PATH)
endif()

if (DEFINED CMAKE_INSTALL_PREFIX)
    file(TO_CMAKE_PATH ${CMAKE_INSTALL_PREFIX} CMAKE_INSTALL_PREFIX)
endif()

if (UNIX AND NOT APPLE)
    include(GNUInstallDirs)
elseif(NOT DEFINED CMAKE_INSTALL_LIBDIR)
    set(CMAKE_INSTALL_LIBDIR "lib")
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_PREFIX_PATH}/${CMAKE_INSTALL_LIBDIR}/cmake")

if (${CMAKE_INSTALL_LIBDIR} STREQUAL "lib64")
    set(FIND_LIBRARY_USE_LIB64_PATHS true)
endif()

if (NOT WIN32 AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo")
endif()

# This is required in order to append /lib/cmake to each element in CMAKE_PREFIX_PATH
set(AWS_MODULE_DIR "/${CMAKE_INSTALL_LIBDIR}/cmake")
string(REPLACE ";" "${AWS_MODULE_DIR};" AWS_MODULE_PATH "${CMAKE_PREFIX_PATH}${AWS_MODULE_DIR}")
# Append that generated list to the module search path
list(APPEND CMAKE_MODULE_PATH ${AWS_MODULE_PATH})

if (CB_BUILD_DEPS)
    file(MAKE_DIRECTORY ${CB_DEPDIR})

    # build aws-c-* dependencies
    if (CB_BUILD_DEPS_AWS)
        #set(BUILD_TESTING OFF)
        #set(STATIC_CRT ON)
        set(BUILD_SHARED_LIBS OFF)
        cb_add_git_dependency(aws-c-common)

        list(APPEND CMAKE_MODULE_PATH "${CB_DEPDIR}/aws-c-common/cmake")
        message(STATUS "CMAKE_MODULE_PATH = ${CMAKE_MODULE_PATH}")
        include(AwsFindPackage)

        set(IN_SOURCE_BUILD ON)

        if (UNIX AND NOT APPLE)
            cb_add_git_dependency(s2n)
        endif()
        cb_add_git_dependency(aws-c-io)
        cb_add_git_dependency(aws-c-cal)
        cb_add_git_dependency(aws-c-compression)
        cb_add_git_dependency(aws-c-http)
        cb_add_git_dependency(aws-c-auth)
    endif()

    # build wxWidgets dependencies
    if (CB_BUILD_DEPS_WX)
#        set(wxBUILD_SHARED OFF)
        #set(wxBUILD_USE_STATIC_RUNTIME ON)
        #set(wxBUILD_MONOLITHIC ON)
 #       set(wxUSE_NO_RTTI ON)
        cb_add_git_dependency(wxWidgets)
    endif()
else()
    include(AwsFindPackage)
    set(IN_SOURCE_BUILD OFF)
endif()

include(AwsCFlags)
include(AwsSharedLibSetup)
include(AwsSanitizers)

# Native CRT bindings files
file(GLOB AWS_CRT_DOTNET_HEADERS src/*.h)
file(GLOB AWS_CRT_DOTNET_SRC src/*.c)

# shared lib that contains the CRT and API bindings, to be loaded by .NET
add_library(${PROJECT_NAME} SHARED ${AWS_CRT_DOTNET_HEADERS} ${AWS_CRT_DOTNET_SRC})

# AWS Common Runtime dependencies
aws_use_package(aws-c-auth)

target_link_libraries(${PROJECT_NAME} ${DEP_AWS_LIBS})

set(TARGET_LIB_DIR "lib")
if (CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(TARGET_ARCH "x64")
else()
    set(TARGET_ARCH "x86")
endif()

set_target_properties(${PROJECT_NAME} PROPERTIES LINKER_LANGUAGE C)
set_target_properties(${PROJECT_NAME} PROPERTIES OUTPUT_NAME "${PROJECT_NAME}-${TARGET_ARCH}")
# for non-msvc, library output will control the shared lib output
set_target_properties(${PROJECT_NAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${TARGET_LIB_DIR})
# for msvc, runtime controls where dlls are output
set_target_properties(${PROJECT_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${TARGET_LIB_DIR})
set_target_properties(${PROJECT_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_DEBUG ${TARGET_LIB_DIR})
set_target_properties(${PROJECT_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELEASE ${TARGET_LIB_DIR})
set_target_properties(${PROJECT_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${TARGET_LIB_DIR})
set_target_properties(${PROJECT_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${TARGET_LIB_DIR})

set(CMAKE_C_FLAGS_DEBUGOPT "")
set_property(TARGET ${PROJECT_NAME} PROPERTY C_STANDARD 99)

aws_set_common_properties(${PROJECT_NAME})
target_include_directories(${PROJECT_NAME} PRIVATE src)

# This is always a shared lib, so always export
target_compile_definitions(${PROJECT_NAME} PUBLIC "-DAWS_DOTNET_USE_IMPORT_EXPORT")
target_compile_definitions(${PROJECT_NAME} PRIVATE "-DAWS_DOTNET_EXPORTS")

target_include_directories(${PROJECT_NAME} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>)
