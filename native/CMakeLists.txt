cmake_minimum_required(VERSION 3.1)
project(catbird LANGUAGES C CXX)

option(BUILD_DEPS "Builds runtime dependencies as part of this project's build" ON)
option(BUILD_DEPS_AWS "Builds aws-c-* runtime dependencies as part of this project's build" ON)
option(BUILD_DEPS_WX "Builds wxWidgets runtime dependencies as part of this project's build" ON)
if (NOT BUILD_DEPS)
    set(BUILD_DEPS_AWS OFF)
    set(BUILD_DEPS_WX OFF)
endif()

if (POLICY CMP0069)
    cmake_policy(SET CMP0069 NEW) # Enable LTO/IPO if available in the compiler, see AwsCFlags
endif()

if (POLICY CMP0077)
    cmake_policy(SET CMP0077 NEW) # Enable options to get their values from normal variables
endif()

macro(add_project_dependency)
    if(NOT EXISTS ${ARGV0})
        execute_process(COMMAND git clone https://github.com/c6burns/${ARGV0} WORKING_DIRECTORY ../deps/)
    endif()
    add_subdirectory(../deps/${ARGV0} ${CMAKE_BINARY_DIR}/deps/${ARGV0})
endmacro()

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

if (BUILD_DEPS)
    file(MAKE_DIRECTORY ../deps)

    # build aws-c-* dependencies
    if (BUILD_DEPS_AWS)
        add_project_dependency(aws-c-common)

        list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/../deps/aws-c-common/cmake")
        message(STATUS "CMAKE_MODULE_PATH = ${CMAKE_MODULE_PATH}")
        include(AwsFindPackage)

        set(IN_SOURCE_BUILD ON)
        set(BUILD_TESTING OFF)
        if (UNIX AND NOT APPLE)
            add_project_dependency(s2n)
        endif()
        add_project_dependency(aws-c-io)
        add_project_dependency(aws-c-cal)
        add_project_dependency(aws-c-compression)
        add_project_dependency(aws-c-http)
        add_project_dependency(aws-c-auth)
    endif()

    # build wxWidgets dependencies
    if (BUILD_DEPS_WX)
        option(wxBUILD_SHARED OFF)
        option(wxBUILD_USE_STATIC_RUNTIME ON)
        option(wxBUILD_MONOLITHIC ON)
        option(wxUSE_NO_RTTI ON)
        option(wxUSE_LIBSDL OFF)
        option(wxUSE_STL OFF)
        option(wxUSE_LIBLZMA OFF)
        add_project_dependency(wxWidgets)
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