# CMake build system for exiv2 library and executables
# Copyright 2015-     Daniel Kaneider  <danielkaneider@users.sf.net>
# Copyright 2010-2012 Gilles Caulier   <caulier dot gilles at gmail dot com>
# Copyright 2008      Patrick Spendrin <ps_ml at gmx dot de>

# Redistribution and use is allowed according to the terms of the BSD license.
# For details see the accompanying COPYING-CMAKE-SCRIPTS file.

##
# Running cmake in a tree in which we have run ./configure ; make
# creates include/exiv2/exv_conf.h which conflicts with cmake's own exv_conf.h
# This causes incorrect compilation and linking errors.
#
# It's OK to delete this as it will be recreated on demand by <exiv2-dir>/Makefile
if( EXISTS "${CMAKE_SOURCE_DIR}/include/exiv2/exv_conf.h"  )
    file(REMOVE "${CMAKE_SOURCE_DIR}/include/exiv2/exv_conf.h")
endif()
##

if( EXISTS "${CMAKE_SOURCE_DIR}/xmpsdk/include"  )
    set( ADOBE_XMP_SDK_INCLUDE ON )
else()
    set( ADOBE_XMP_SDK_INCLUDE OFF )
    message( FATAL_ERROR "Missing Adobe XMP SDK include files. Setup cannot proceed." )
endif()

CMAKE_MINIMUM_REQUIRED( VERSION 3.1.0 )
PROJECT( exiv2 )

if( POLICY CMP0042 )
    cmake_policy(SET CMP0042 NEW) # enable MACOSX_RPATH support
else()
    SET(CMAKE_MACOSX_RPATH 1)
endif()

SET( PACKAGE_COPYRIGHT      "Andreas Huggel" )
SET( PACKAGE_BUGREPORT      "http://github.com/exiv2/exiv2" )
SET( PACKAGE                "exiv2" )
SET( PACKAGE_NAME           "exiv2" )
SET( PACKAGE_STRING         "exiv2 0.26" )
SET( PACKAGE_TARNAME        "exiv2" )
SET( PACKAGE_VERSION        "0.26" )
SET( PACKAGE_URL            "http://www.exiv2.org" )
SET( GENERIC_LIB_VERSION    "26.0.0" )
SET( GENERIC_LIB_SOVERSION  "26" )

# options and their default values
OPTION( EXIV2_CPP_STANDARD           "Set the C++ Standard to compile with"                 11  )
OPTION( EXIV2_CPP_STANDARD_REQUIRED  "Set C++ Standard to required"                         ON  )
OPTION( EXIV2_CPP_EXTENSION          "Set C++ Extensions to required"                       OFF )
OPTION( EXIV2_ENABLE_VERBOSITY       "Enable verbosity"                                     OFF )
OPTION( EXIV2_ENABLE_SHARED        "Build exiv2 as a shared library (dll)"                 ON  )
OPTION( EXIV2_ENABLE_XMP           "Build with XMP metadata support"                       ON  )
OPTION( EXIV2_ENABLE_PNG           "Build with png support (requires libz)"                ON  )
OPTION( EXIV2_ENABLE_NLS           "Build native language support (requires gettext)"      ON  )
OPTION( EXIV2_ENABLE_PRINTUCS2     "Build with Printucs2"                                  ON  )
OPTION( EXIV2_ENABLE_LENSDATA      "Build including lens data"                             ON  )
OPTION( EXIV2_ENABLE_COMMERCIAL    "Build with the EXV_COMMERCIAL_VERSION symbol set"      OFF )
OPTION( EXIV2_ENABLE_BUILD_SAMPLES "Build the unit tests"                                  ON  )
OPTION( EXIV2_ENABLE_BUILD_PO      "Build translations files"                              OFF )
OPTION( EXIV2_ENABLE_VIDEO         "Build video support into library"                      OFF )
OPTION( EXIV2_ENABLE_WEBREADY      "Build webready support into library"                   OFF )
IF (WIN32)
    OPTION( EXIV2_ENABLE_DYNAMIC_RUNTIME  "Use dynamic runtime (used for static libs)"     OFF )
    OPTION( EXIV2_ENABLE_WIN_UNICODE   "Use Unicode paths (wstring) on Windows"            OFF )
    OPTION( EXIV2_ENABLE_CURL          "USE Libcurl for HttpIo"                            OFF )
    OPTION( EXIV2_ENABLE_SSH           "USE Libssh for SshIo"                              OFF )
ELSE()
    OPTION( EXIV2_ENABLE_CURL          "USE Libcurl for HttpIo"                            ON  )
    OPTION( EXIV2_ENABLE_SSH           "USE Libssh for SshIo"                              ON  )
    SET ( EXIV2_ENABLE_DYNAMIC_RUNTIME OFF )
ENDIF()

SET ( CMAKE_VERBOSE_MAKEFILE ${EXIV2_ENABLE_VERBOSITY} )
IF ( NOT ${EXIV2_CPP_STANDARD} )
    SET ( EXIV2_CPP_STANDARD 11 )
ENDIF ()

include(config/findDependencies.cmake)

IF ( WIN32 OR MINGW OR MSVC )
    SET( XMPLIB_EXT lib )
ELSE()
    SET( XMPLIB_EXT a )
ENDIF()

IF ( EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/xmpsdk/libXMPCore.${XMPLIB_EXT}" AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/xmpsdk/libXMPFiles.${XMPLIB_EXT}" )
    MESSAGE( STATUS "Has xmp sdk library files" )
    SET ( EXV_HAS_XMP_SDK_LIBS ON )
ELSE()
    MESSAGE( STATUS "Doesn't have xmp sdk library files" )
    SET ( EXV_HAS_XMP_SDK_LIBS OFF )
ENDIF()

IF( MINGW OR UNIX )
    IF ( CMAKE_CXX_COMPILER STREQUAL "g++" OR CMAKE_C_COMPILER STREQUAL "gcc" )
        ADD_DEFINITIONS(-Wall
                    -Wcast-align
                    -Wpointer-arith
                    -Wformat-security
                    -Wmissing-format-attribute
                    -Woverloaded-virtual
                    -W
                   )
    ENDIF()

    execute_process(COMMAND ${CMAKE_CXX_COMPILER} --version OUTPUT_VARIABLE COMPILER_VERSION)
    string(REGEX MATCHALL "[A-Z|a-z\+]+" GCC_COMPILER_COMPONENTS ${COMPILER_VERSION})
    list(GET GCC_COMPILER_COMPONENTS 0 COMPILER)

    execute_process(COMMAND ${CMAKE_CXX_COMPILER} -dumpversion OUTPUT_VARIABLE GCC_VERSION)
    string(REGEX MATCHALL "[0-9]+" GCC_VERSION_COMPONENTS ${GCC_VERSION})
    list(GET GCC_VERSION_COMPONENTS 0 GCC_MAJOR)

ENDIF( MINGW OR UNIX )

set( CMAKE_CXX_STANDARD ${EXIV2_CPP_STANDARD} )
set( CMAKE_CXX_STANDARD_REQUIRED ${EXIV2_CPP_STANDARD_REQUIRED} )
set( CMAKE_CXX_EXTENSIONS ${EXIV2_CPP_EXTENSION} )

IF ( ${EXIV2_CPP_STANDARD} LESS 98 AND ${EXIV2_CPP_STANDARD} GREATER 10  )
    ADD_DEFINITIONS( -DEXV_USING_CPP_ELEVEN=1  )
ENDIF()

IF( EXIV2_ENABLE_COMMERCIAL )
    SET (EXIV2_ENABLE_LENSDATA OFF)
    SET (EXIV2_ENABLE_NLS OFF)
ENDIF()

set (PRIVATE_VAR)
if(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} GREATER 2.8)
    set (PRIVATE_VAR PRIVATE)
endif()

# binary as first, since it workarounds old VS solution compatibility problems
INCLUDE_DIRECTORIES( ${CMAKE_SOURCE_DIR}/src/ ${CMAKE_SOURCE_DIR}/include/  ${CMAKE_SOURCE_DIR}/include/exiv2/)

if( MSVC )
    # cmake_policy(SET CMP0008)
    ADD_DEFINITIONS(-DPSAPI_VERSION=1) # to be compatible with <= WinVista (#905)
    set(CMAKE_DEBUG_POSTFIX "d")
endif( MSVC )

IF( EXIV2_ENABLE_XMP )
    SET( HAVE_XMP_TOOLKIT ON )
    SET( XMPLIB "do" )
    IF (NOT MINGW)
        set(THREADS_PREFER_PTHREAD_FLAG ON)
    ENDIF()
ENDIF( EXIV2_ENABLE_XMP )

INCLUDE( config/CMakeChecks.txt )

SET( INSTALL_TARGET_STANDARD_ARGS
     RUNTIME DESTINATION bin
     LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
     ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

IF ( WIN32 OR MINGW OR MSVC )
    SET( XMPLIB_EXT lib )
ELSE()
    SET( XMPLIB_EXT a )
ENDIF()

IF ( EXV_HAS_XMP_SDK_LIBS )
    SET ( XMPLIBS ${CMAKE_CURRENT_SOURCE_DIR}/xmpsdk/libXMPCore.${XMPLIB_EXT}
                  ${CMAKE_CURRENT_SOURCE_DIR}/xmpsdk/libXMPFiles.${XMPLIB_EXT}
)
ENDIF()

##
# Adobe XMP SDK definitions
IF( MSVC OR WIN32 OR MINGW)
  ADD_DEFINITIONS( -DXMP_WinBuild=1 -DWIN_ENV=1 )
ELSEIF( UNIX AND APPLE) 
  ADD_DEFINITIONS( -DXMP_MacBuild=1 -DMAC_ENV=1 )
  SET(CMAKE_OSX_SYSROOT=/Developer/SDKs/MacOSX10.12.sdk/)
  SET(CMAKE_OSX_DEPLOYMENT_TARGET "10.9")
ELSE()
  ADD_DEFINITIONS( -DXMP_UNIXBuild=1 -DUNIX_ENV=1 )
ENDIF()

# the following lines need to be added to circumvent the error in xmpsdk\src\MD5.h
IF( ${EXV_HAVE_STDINT_H} )
    ADD_DEFINITIONS( -DEXV_HAVE_STDINT_H )
ENDIF( ${EXV_HAVE_STDINT_H} )

ADD_SUBDIRECTORY( src )

IF ( EXV_HAS_XMP_SDK_LIBS )
IF( EXIV2_ENABLE_BUILD_SAMPLES )
    ADD_SUBDIRECTORY( samples )
ENDIF( EXIV2_ENABLE_BUILD_SAMPLES )
ENDIF()

IF( EXIV2_ENABLE_BUILD_PO )
    ADD_SUBDIRECTORY( po )
ENDIF( EXIV2_ENABLE_BUILD_PO )

##
# tests
IF ( EXV_HAS_XMP_SDK_LIBS )
ADD_CUSTOM_TARGET(tests       COMMAND env EXIV2_BINDIR="${CMAKE_BINARY_DIR}"/bin make tests       WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" )
ADD_CUSTOM_TARGET(geotag-test COMMAND env EXIV2_BINDIR="${CMAKE_BINARY_DIR}"/bin make geotag-test WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" )
ENDIF()

##
# http://dev.exiv2.org/boards/3/topics/1364
# effectively does a make doc on the root directory
# has to run 'make config' and './configure'
# and copy bin/taglist to <exiv2dir>/bin/taglist for use by 'make doc'
IF( MINGW OR UNIX OR APPLE)
  ADD_CUSTOM_TARGET(doc
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/doc"
    COMMAND chmod +x ./cmake_doc.sh
    COMMAND ./cmake_doc.sh "${CMAKE_BINARY_DIR}"
  )
ENDIF()

INCLUDE( config/printSummary.cmake )
# That's all Folks!
##
