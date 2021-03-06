cmake_minimum_required(VERSION 3.1)
project(aws-crt-nodejs C)
option(BUILD_DEPS "Builds aws common runtime dependencies as part of build, only do this if you don't want to control your dependency chain." ON)

include(CTest)

# ensure that the release build has symbols
if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif()

if (UNIX AND NOT APPLE)
    include(GNUInstallDirs)
elseif(NOT DEFINED CMAKE_INSTALL_LIBDIR)
    set(CMAKE_INSTALL_LIBDIR "lib")
endif()

if (${CMAKE_INSTALL_LIBDIR} STREQUAL "lib64")
    set(FIND_LIBRARY_USE_LIB64_PATHS true)
endif()

# This is required in order to append /lib/cmake to each element in CMAKE_PREFIX_PATH
set(AWS_MODULE_DIR "/${CMAKE_INSTALL_LIBDIR}/cmake")
string(REPLACE ";" "${AWS_MODULE_DIR};" AWS_MODULE_PATH "${CMAKE_PREFIX_PATH}${AWS_MODULE_DIR}")
# Append that generated list to the module search path
list(APPEND CMAKE_MODULE_PATH ${AWS_MODULE_PATH})

if (BUILD_DEPS)
    message(STATUS "Using submodule dependencies")
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/aws-common-runtime/aws-c-common/cmake")
    include(AwsFindPackage)

    set(IN_SOURCE_BUILD ON)
    set(BUILD_TESTING_PREV ${BUILD_TESTING})
    set(BUILD_TESTING OFF)

    add_subdirectory(aws-common-runtime/aws-c-common)
    if (UNIX AND NOT APPLE)
        list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/aws-common-runtime/s2n/cmake/modules")
        find_package(LibCrypto REQUIRED)
        add_subdirectory(aws-common-runtime/s2n)
    endif()

    add_subdirectory(aws-common-runtime/aws-c-io)
    add_subdirectory(aws-common-runtime/aws-c-cal)
    add_subdirectory(aws-common-runtime/aws-c-compression)
    add_subdirectory(aws-common-runtime/aws-c-http)
    add_subdirectory(aws-common-runtime/aws-c-auth)
    add_subdirectory(aws-common-runtime/aws-c-mqtt)
    
    set(BUILD_TESTING ${BUILD_TESTING_PREV})
else()
    include(AwsFindPackage)
    set(IN_SOURCE_BUILD OFF)
endif()

if (POLICY CMP0069)
    cmake_policy(SET CMP0069 NEW) # Enable LTO/IPO if available in the compiler, see AwsCFlags
endif()

set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_SOURCE_DIR}/dist")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_PREFIX_PATH}/${CMAKE_INSTALL_LIBDIR}/cmake")

include(AwsCFlags)
include(AwsSharedLibSetup)
include(AwsSanitizers)

file(GLOB AWS_CRT_SRC
       "source/*.c"
)

add_library(${PROJECT_NAME} SHARED ${AWS_CRT_SRC})
aws_set_common_properties(${PROJECT_NAME})
aws_prepare_symbol_visibility_args(${PROJECT_NAME} "AWS_CRT_NODEJS")
aws_add_sanitizers(${PROJECT_NAME})

# Gives our library file a .node extension without any "lib" prefix
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "" SUFFIX ".node")

target_include_directories(${PROJECT_NAME} PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
        ${CMAKE_JS_INC})

aws_use_package(aws-c-http REQUIRED)
aws_use_package(aws-c-mqtt REQUIRED)
aws_use_package(aws-c-auth REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE ${CMAKE_JS_LIB} ${DEP_AWS_LIBS})

if (CMAKE_JS_PLATFORM AND CMAKE_JS_ARCH)
    # When run from build.ts, these are set
    set(destination bin/${CMAKE_JS_PLATFORM}-${CMAKE_JS_ARCH})
else()
    # If not, default to "native"
    set(destination bin/native)
endif()

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_BUILD_TYPE}/aws-crt-nodejs.node"
    DESTINATION ${destination})
