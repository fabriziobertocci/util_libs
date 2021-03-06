#
# Copyright 2018, Data61
# Commonwealth Scientific and Industrial Research Organisation (CSIRO)
# ABN 41 687 119 230.
#
# This software may be distributed and modified according to the terms of
# the BSD 2-Clause license. Note that NO WARRANTY is provided.
# See "LICENSE_BSD2.txt" for details.
#
# @TAG(DATA61_BSD)
#

cmake_minimum_required(VERSION 3.7.2)
include(ExternalProject)
project(libpicotcp NONE)

config_option(LibPicotcp LIB_PICOTCP
    "Build libpicotcp"
    DEFAULT OFF
)
config_option(LibPicotcpBsd LIB_PICOTCP_BSD
    "Build libpicotcp with bsd socket support"
    DEFAULT OFF
    DEPENDS LibPicotcp)

if(LibPicotcp)
    # find pico tcp location
    find_file(PICOTCP_PATH picotcp PATHS ${CMAKE_SOURCE_DIR}/projects CMAKE_FIND_ROOT_PATH_BOTH)
    mark_as_advanced(FORCE PICOTCP_PATH)
    if("${PICOTCP_PATH}" STREQUAL "PICOTCP_PATH-NOTFOUND")
        message(FATAL_ERROR "Failed to find picotcp. Consider cmake -DPICOTCP_PATH=/path/to/picotcp")
    endif()

    # extract compiler args from cmake
    get_property(compile_options DIRECTORY PROPERTY COMPILE_OPTIONS)
    separate_arguments(cmake_c_flags_sep NATIVE_COMMAND "${CMAKE_C_FLAGS}")
    list(APPEND compile_options "${cmake_c_flags_sep}")

    # add the location of muslc headers to the cflags
    string(APPEND compile_options " -I$<JOIN:$<TARGET_PROPERTY:muslc,INTERFACE_INCLUDE_DIRECTORIES>, -I>")

    # generate a build.sh script to avoid nasty escape issues, we then invoke this in ExternalProject_Add
    file(GENERATE OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/build.sh" CONTENT "${CMAKE_COMMAND} -E env \
        \"PLATFORM_CFLAGS=$<JOIN:${compile_options}, >\" \
        \"CROSS_COMPILE=${CROSS_COMPILER_PREFIX}\" \
        make -s -j")

    # build picotcp using its very own build system
    ExternalProject_Add(picotcp_external
        URL file:///${PICOTCP_PATH}
        CONFIGURE_COMMAND "true"
        BUILD_COMMAND sh "${CMAKE_CURRENT_BINARY_DIR}/build.sh"
        BUILD_IN_SOURCE 1
        INSTALL_COMMAND "true"
        EXCLUDE_FROM_ALL
        BUILD_ALWAYS ON
        DEPENDS muslc
        BUILD_BYPRODUCTS "<SOURCE_DIR>/build/lib/libpicotcp.a"
    )

    # get the dir that the picotcp sources have been copied to by ExternalProject_Add
    ExternalProject_Get_Property(picotcp_external SOURCE_DIR)

    # create a library which is the sources
    add_library(picotcp_sources STATIC IMPORTED GLOBAL)
    add_dependencies(picotcp_sources picotcp_external)
    set_property(TARGET picotcp_sources PROPERTY IMPORTED_LOCATION "${SOURCE_DIR}/build/lib/libpicotcp.a")

    # create a library which is the header files, that depends on the sources
    add_library(picotcp INTERFACE)
    add_dependencies(picotcp picotcp_sources)
    set_property(TARGET picotcp PROPERTY INTERFACE_LINK_LIBRARIES picotcp_sources)
    target_include_directories(picotcp INTERFACE "${SOURCE_DIR}/build/include")
endif()
if(LibPicotcpBsd)
    # find pico tcp location
    find_file(PICOTCP_BSD_PATH picotcp-bsd PATHS ${CMAKE_SOURCE_DIR}/projects CMAKE_FIND_ROOT_PATH_BOTH)
    mark_as_advanced(FORCE PICOTCP_BSD_PATH)
    if("${PICOTCP_BSD_PATH}" STREQUAL "PICOTCP_BSD_PATH-NOTFOUND")
        message(FATAL_ERROR "Failed to find picotcp-bsd. Consider cmake -DPICOTCP_BSD_PATH=/path/to/picotcp-bsd")
    endif()

    add_definitions(-DSTDSOCKET)
    add_definitions(-U__linux__)
    add_definitions(-D_SYS_POLL_H)

    # create picotcp_bsd library
    add_library(picotcp_bsd STATIC EXCLUDE_FROM_ALL ${PICOTCP_BSD_PATH}/pico_bsd_sockets.c
                                                    ${PICOTCP_BSD_PATH}/pico_osal_noos.c)
    target_include_directories(picotcp_bsd PUBLIC ${PICOTCP_BSD_PATH})
    target_link_libraries(picotcp_bsd muslc picotcp)
endif()
