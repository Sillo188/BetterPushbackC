# CDDL HEADER START
#
# This file and its contents are supplied under the terms of the
# Common Development and Distribution License ("CDDL"), version 1.0.
# You may only use this file in accordance with the terms of version
# 1.0 of the CDDL.
#
# A full copy of the text of the CDDL should have accompanied this
# source.  A copy of the CDDL is also available via the Internet at
# http://www.illumos.org/license/CDDL.
#
# CDDL HEADER END

# Copyright 2017 Saso Kiselkov. All rights reserved.

cmake_minimum_required(VERSION 2.8)
project(bp C)

SET(SRC acf_outline.c bp.c bp_cam.c cab_view.c cfg.c driving.c ff_a320_intf.c
    msg.c tug.c wed2route.c xplane.c)
SET(HDR acf_outline.h bp.h bp_cam.h cab_view.h cfg.h driving.h ff_a320_intf.h
    msg.h tug.h wed2route.h xplane.h)

SET(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELEASE} -DDEBUG")
SET(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DDEBUG -O0")

SET(ALL_SRC ${SRC} ${HDR})
LIST(SORT ALL_SRC)

if(APPLE)
	add_executable(bp ${ALL_SRC})
else()
	add_library(bp SHARED ${ALL_SRC})
endif()

include_directories(bp PUBLIC "${LIBACFUTILS}/src")

#compiler flags
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror --std=c99 \
    -Wno-unused-local-typedefs")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} \
    -DCHECK_RESULT_USED=\"__attribute__ ((warn_unused_result))\"")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_GNU_SOURCE -D_FILE_OFFSET_BITS=64")

#libraries
if(UNIX)
	include_directories(bp PUBLIC "../OpenAL/include")
	include_directories(bp PUBLIC "${LIBACFUTILS}/SDK/CHeaders/XPLM"
	    "${LIBACFUTILS}/SDK/CHeaders/Widgets"
	    "${LIBACFUTILS}/SDK/CHeaders/Wrappers"
	    "${LIBACFUTILS}/acf_apis")
endif(UNIX)
if(APPLE)
	include_directories(bp PUBLIC "../libpng/libpng-mac-64/include")
	include_directories(bp PUBLIC "../pcre2/pcre2-mac-64/include")
	include_directories(bp PUBLIC
	    "../libxml2/libxml2-mac-64/include/libxml2")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -arch x86_64 -Wno-deprecated")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mmacosx-version-min=10.7")
elseif(UNIX)
	include_directories(bp PUBLIC "../libpng/libpng-linux-64/include")
	include_directories(bp PUBLIC "../pcre2/pcre2-linux-64/include")
	include_directories(bp PUBLIC
	    "../libxml2/libxml2-linux-64/include/libxml2")
endif()

# Yes, we need to strip a trailing newline from command output. CMake is
# "smart" like that...
execute_process(COMMAND git rev-parse --short HEAD
    OUTPUT_VARIABLE BP_PLUGIN_VERSION)
string(REGEX REPLACE "\n$" "" BP_PLUGIN_VERSION "${BP_PLUGIN_VERSION}")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DBP_PLUGIN_VERSION='\"${BP_PLUGIN_VERSION}\"'")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DXPLM200=1 -DXPLM210=1")
if(APPLE)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DAPL=1 -DIBM=0 -DLIN=0")
else()
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DAPL=0 -DIBM=0 -DLIN=1")
endif()

# linking
# Careful: order matters in target_link_libraries. Dependencies must be
# ordered as first symbol callers and then as symbol providers (i.e.
# libpng16 depends on libz, so linking must be first: libpng, then libz).
link_libraries(m)
set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH}
    ../zlib/zlib-linux-64 ../libpng/libpng-linux-64/.libs
    ../pcre2/pcre2-linux-64/.libs
    ${LIBACFUTILS}/qmake/lin64
    ${LIBACFUTILS}/opus/opusfile-linux-64/lib
    ${LIBACFUTILS}/opus/opus-linux-64/lib
    ${LIBACFUTILS}/opus/libogg-linux-64/install/lib
    ../libxml2/libxml2-linux-64/.libs)
FIND_LIBRARY(LIBACFUTILS_LIBRARY acfutils)
FIND_LIBRARY(LIBPNG_LIBRARY png16)
FIND_LIBRARY(ZLIB_LIBRARY z)
FIND_LIBRARY(PCRE2_LIBRARY pcre2-8)
FIND_LIBRARY(OPUSFILE_LIBRARY opusfile)
FIND_LIBRARY(OPUS_LIBRARY opus)
FIND_LIBRARY(OGG_LIBRARY ogg)
FIND_LIBRARY(LIBXML2_LIBRARY xml2)
FIND_LIBRARY(LIBC_NONSHARED c_nonshared)
target_link_libraries(bp
    ${LIBACFUTILS_LIBRARY}
    ${LIBPNG_LIBRARY}
    ${ZLIB_LIBRARY}
    ${PCRE2_LIBRARY}
    ${OPUSFILE_LIBRARY}
    ${OPUS_LIBRARY}
    ${OGG_LIBRARY}
    ${LIBXML2_LIBRARY}
    ${LIBC_NONSHARED}
    )

SET_TARGET_PROPERTIES(bp PROPERTIES PREFIX "")
SET_TARGET_PROPERTIES(bp PROPERTIES SUFFIX "")

if(APPLE)
	set(OUTPUT_FILENAME "mac.xpl")
	execute_process(COMMAND ${LIBACFUTILS}/pkg-config-deps mac-64 --libs
	    OUTPUT_VARIABLE LIBACFUTILS_DEP_LDFLAGS)
	SET_TARGET_PROPERTIES(bp PROPERTIES LINK_FLAGS
	    "${CMAKE_SHARED_LINKER_FLAGS} -fPIC -fvisibility=hidden -bundle")
else()
	set(OUTPUT_FILENAME "lin.xpl")
	execute_process(COMMAND ${LIBACFUTILS}/pkg-config-deps linux-64 --libs
	    OUTPUT_VARIABLE LIBACFUTILS_DEP_LDFLAGS)
	SET_TARGET_PROPERTIES(bp PROPERTIES LINK_FLAGS
	    "${CMAKE_SHARED_LINKER_FLAGS} -rdynamic -nodefaultlibs \
	    -undefined_warnsing -fPIC -fvisibility=hidden")
endif()

set_target_properties(bp PROPERTIES RUNTIME_OUTPUT_DIRECTORY
    "${CMAKE_SOURCE_DIR}/bin" )
set_target_properties(bp PROPERTIES LIBRARY_OUTPUT_DIRECTORY
    "${CMAKE_SOURCE_DIR}/bin" )
set_target_properties(bp PROPERTIES OUTPUT_NAME "${OUTPUT_FILENAME}")
