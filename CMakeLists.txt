###############################################################################
#
# CMake main configuration file to build GeoTIFF library and utilities.
#
# Author: Mateusz Loskot <mateusz@loskot.net>
# Author: Dmitry Baryshnikov <polimax@mail.ru>
#
###############################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)
    
project(geotiff C)

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

include(GNUInstallDirs)

set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
set(INSTALL_SHARE_DIR ${CMAKE_INSTALL_DATADIR} CACHE INTERNAL "Shared files directory" FORCE)
set(INSTALL_DOC_DIR ${CMAKE_INSTALL_DOCDIR} CACHE INTERNAL "Doc files directory" FORCE)

if(CMAKE_GENERATOR_TOOLSET MATCHES "v([0-9]+)_xp")
    add_definitions(-D_WIN32_WINNT=0x0501)
endif()

include(util)

# Version information
check_version(PROJECT_VERSION_MAJOR PROJECT_VERSION_MINOR PROJECT_VERSION_PATCH PROJECT_VERSION_REV)

set(VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")
set(GeoTIFF_VERSION ${VERSION})

# Set library version to match that of autoconf:
# libgeotiff.so -> libgeotiff.so.2.1.1
# libgeotiff.so.2 -> libgeotiff.so.2.1.1
# libgeotiff.so.2.1.1
set(LINK_SOVERSION "2")
set(LINK_VERSION "2.1.1")

report_version(${PROJECT_NAME} "${VERSION} [${LINK_VERSION}]")  

###############################################################################
# General build settings

set(GEOTIFF_BUILD_PEDANTIC FALSE CACHE BOOL "Choose compilation in pedantic or relaxed mode")
if(CMAKE_BUILD_TYPE MATCHES Debug)
    set(GEOTIFF_BUILD_PEDANTIC TRUE)
endif()

#if (CMAKE_MAJOR_VERSION GREATER 2)
#    cmake_policy(SET CMP0022 OLD) # interface link libraries
#    cmake_policy(SET CMP0042 OLD) # osx rpath
#endif()

###############################################################################
# Platform and compiler specific settings

if(WIN32)
    if(MSVC)
        add_definitions(-DBUILD_AS_DLL=1)
        add_definitions(/DW4)
        if (NOT (MSVC_VERSION VERSION_LESS 1400))
            add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
            add_definitions(-D_CRT_SECURE_NO_WARNINGS)
            add_definitions(-D_CRT_NONSTDC_NO_WARNING)
            add_definitions(-D_SCL_SECURE_NO_WARNINGS)
        endif()
    endif(MSVC)
endif()

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
    set(COMPILE_FLAGS "-fPIC -Wall -Wno-long-long -Wno-implicit-function-declaration -Wno-int-conversion")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${COMPILE_FLAGS} -std=c99")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${COMPILE_FLAGS} -std=c++98")
    if(GEOTIFF_BUILD_PEDANTIC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pedantic")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic")
    endif()
endif()

# Check required standard headers
include(CheckIncludeFiles)
check_include_files(stdio.h HAVE_STDIO_H)
check_include_files(stdlib.h HAVE_STDLIB_H)
check_include_files(string.h HAVE_STRING_H)

configure_file(${CMAKE_MODULE_PATH}/cmake_uninstall.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake IMMEDIATE @ONLY)

###############################################################################
# User-defined build settings

set(GEOTIFF_CSV_NAMES area.csv codes.csv datum.csv gcs.csv pcs.csv)
find_path(GEOTIFF_CSV_DATA_DIR NAMES ${GEOTIFF_CSV_NAMES}
    PATHS "${CMAKE_SOURCE_DIR}/csv"
    DOC "Default location of GeoTIFF CSV files"
    NO_DEFAULT_PATH)

if(IS_DIRECTORY ${GEOTIFF_CSV_DATA_DIR} AND EXISTS "${GEOTIFF_CSV_DATA_DIR}/gcs.csv")
    message(STATUS "Found GeoTIFF CSV files in: ${GEOTIFF_CSV_DATA_DIR}")
    add_definitions(-DCSV_DATA_DIR="${GEOTIFF_CSV_DATA_DIR}")
else()
    message(FATAL_ERROR "Failed to find GeoTIFF CSV files in: ${GEOTIFF_CSV_DATA_DIR}")
endif()

###############################################################################
# Search for dependencies

include_directories(${PROJECT_SOURCE_DIR})
include_directories(${PROJECT_SOURCE_DIR}/libxtiff)

include(FindAnyProject)

# TIFF support - required, default=ON
find_anyproject(TIFF REQUIRED)
if(TIFF_FOUND)
    add_definitions(-DHAVE_TIFF=1)
endif(TIFF_FOUND)

# PROJ.4 support - optional, default=ON
find_anyproject(PROJ4)
if(PROJ4_FOUND)
    add_definitions(-DHAVE_LIBPROJ=1)
endif()

# Zlib support - optional, default=OFF
find_anyproject(ZLIB DEFAULT OFF)
if(ZLIB_FOUND)
    add_definitions(-DHAVE_ZIP=1)
endif()

# JPEG support - optional, default=OFF
find_anyproject(JPEG DEFAULT OFF)
if(JPEG_FOUND)
    add_definitions(-DHAVE_JPEG=1)
endif()

# Turn off TOWGS84 support
set(WITH_TOWGS84 TRUE CACHE BOOL "Build with TOWGS84 support")
if (NOT WITH_TOWGS84)
   set(GEO_NORMALIZE_DISABLE_TOWGS84 1)
endif()

###############################################################################
# Generate geo_config.h with compile-time configuration

configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/geo_config.h.in geo_config.h)
include_directories ( ${CMAKE_CURRENT_BINARY_DIR} )


###############################################################################
# Installation settings

set(GEOTIFF_CSV_DATA
    csv/alias.csv
    csv/area.csv
    csv/change.csv
    csv/compdcs.csv
    csv/coordinate_axis.csv
    csv/coordinate_axis_name.csv
    csv/coordinate_operation.csv
    csv/coordinate_operation_method.csv
    csv/coordinate_operation_parameter.csv
    csv/coordinate_operation_parameter_value.csv
    csv/coordinate_operation_path.csv
    csv/coordinate_reference_system.csv
    csv/coordinate_system.csv
    csv/datum.csv
    csv/datum_shift.csv
    csv/datum_shift_pref.csv
    csv/deprecation.csv
    csv/ellipsoid.csv
    csv/esri_datum_override.csv
    csv/gcs.csv
    csv/gcs.override.csv
    csv/geoccs.csv
    csv/naming_system.csv
    csv/pcs.csv
    csv/pcs.override.csv
    csv/prime_meridian.csv
    csv/projop_wparm.csv
    csv/stateplane.csv
    csv/supersession.csv
    csv/unit_of_measure.csv
    csv/version_history.csv
    csv/vertcs.csv
    csv/vertcs.override.csv )

set(GEOTIFF_LIB_HEADERS
    geotiff.h
    geotiffio.h
    geovalues.h
    geonames.h
    geokeys.h
    ${CMAKE_CURRENT_BINARY_DIR}/geo_config.h
    geo_tiffp.h
    geo_keyp.h
    geo_normalize.h
    cpl_serv.h
    geo_simpletags.h
    epsg_datum.inc
    epsg_gcs.inc
    epsg_pm.inc
    epsg_units.inc
    geo_ctrans.inc
    epsg_ellipse.inc
    epsg_pcs.inc
    epsg_proj.inc
    epsg_vertcs.inc
    geokeys.inc
    libxtiff/xtiffio.h)

# Install doc files
install(FILES
    AUTHORS ChangeLog COPYING INSTALL LICENSE README README_BIN README.WIN
    DESTINATION "${INSTALL_DOC_DIR}")

# Install CSV data files
install(FILES ${GEOTIFF_CSV_DATA} DESTINATION ${INSTALL_SHARE_DIR}/epsg_csv)

# Install header files for development distribution
install(FILES ${GEOTIFF_LIB_HEADERS} DESTINATION ${INSTALL_INC_DIR})

set(EXPORT_TARGETS)

###############################################################################
# Build libxtiff library

add_subdirectory(libxtiff)

###############################################################################
# Build libgeotiff library

set(GEOTIFF_LIB_SOURCES
    cpl_serv.c
    cpl_csv.c
    geo_extra.c
    geo_free.c
    geo_get.c
    geo_names.c
    geo_new.c
    geo_normalize.c
    geo_print.c
    geo_set.c
    geo_simpletags.c
    geo_tiffp.c
    geo_trans.c
    geo_write.c
    geotiff_proj4.c)

set(INCODE_EPSG_FROM_CSV FALSE CACHE BOOL "Build source-encoded EPSG data files")

foreach(epsg_csv_file ${GEOTIFF_CSV_DATA})
    # We cannot use NAME_WE without string-replace, because that would
    # give "gcs.csv" and "gcs.override.csv" the same epsg_csv_name
    get_filename_component(epsg_csv_name "${epsg_csv_file}" NAME)
    string(REPLACE ".csv" "" epsg_csv_basename "${epsg_csv_name}")
    # SET(INCODE_EPSG_FROM_CSV_${epsg_csv_basename} FALSE CACHE BOOL "Build ${epsg_csv_basename} file target")

    # IF(INCODE_EPSG_FROM_CSV_${epsg_csv_basename})
    if( INCODE_EPSG_FROM_CSV)
        set(epsg_source_name "${CMAKE_CURRENT_BINARY_DIR}/incode_epsg_${epsg_csv_basename}.c")
        list(APPEND GEOTIFF_LIB_CSV_SOURCES "${epsg_source_name}")

        # sanitize .'s out of struct names
        string(REGEX REPLACE "\\." "_" STRUCT_NAME ${epsg_csv_basename})
        list(APPEND epsg_includefile_externconst "\nextern const datafile_rows_t *${STRUCT_NAME}_rows[]")
        list(APPEND epsg_includefile_pointer_list "  { \"${STRUCT_NAME}\", ${STRUCT_NAME}_rows },")
        message(STATUS "Creating buildrule to convert ${epsg_csv_basename}.csv to ${epsg_csv_basename}.c and include it in code.")
        add_custom_command(OUTPUT "${epsg_source_name}"
            COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/csv/csv2c.py" "${CMAKE_CURRENT_SOURCE_DIR}/${epsg_csv_file}" "${epsg_source_name}"
            DEPENDS "${epsg_csv_file}"
            WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
            VERBATIM)
    endif()
endforeach()

set(EPSG_INCODE_EXPLANATION
    "This file is autogenerated by CMake, based on the INCODE_EPSG_* options specified during configure.\n
    Choosing an EPSG CSV file for inclusion into code will run csv/csv2c.py on the file and include the\n
    resulting c-source as a build target, as opposed to using GDAL_DATA-dir to find the EPSG codes at runtime.")
string(REPLACE ";" "\n" EPSG_INCLUDEFILE_POINTER_STRING "${epsg_includefile_pointer_list}")
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/epsg_incode_header.h"
    "/* ${EPSG_INCODE_EXPLANATION} */\n${epsg_includefile_externconst}; \n\n/* Pointers to csv data included in code */\nstatic const datafile_t files[] = {\n${EPSG_INCLUDEFILE_POINTER_STRING}\n  { NULL, NULL }};")

set(XTIFF_SOURCES libxtiff/xtiff.c)

#---
# Static libgeotiff archive
# NOTE: Did not put XTIFF_SOURCES in static lib because libxtiff.a is written out
#       currently.
#---
if (MSVC OR CMAKE_CONFIGURATION_TYPES)
  # For multi-config systems and for Visual Studio, the debug versions
  # of the libraries have a _d suffix.
  set (CMAKE_DEBUG_POSTFIX _d)
endif ()

if(BUILD_SHARED_LIBS)
    set(LIB_NAME ${PROJECT_NAME})
    add_library(${LIB_NAME} SHARED ${GEOTIFF_LIB_SOURCES} ${GEOTIFF_LIB_CSV_SOURCES} ${XTIFF_SOURCES})
    
    # Windows:
    if(WIN32 AND MSVC)
        set_target_properties(${LIB_NAME} PROPERTIES IMPORT_SUFFIX "_i.lib")
    endif(WIN32 AND MSVC)
else()    
    set(LIB_NAME ${PROJECT_NAME}static)
    add_library(${LIB_NAME} STATIC ${GEOTIFF_LIB_SOURCES} ${GEOTIFF_LIB_CSV_SOURCES})
endif()  

set(EXPORT_TARGETS ${EXPORT_TARGETS} ${LIB_NAME}) 

target_link_extlibraries(${LIB_NAME}) 

export(TARGETS ${EXPORT_TARGETS} FILE ${LIB_NAME}-exports.cmake EXPORT_LINK_INTERFACE_LIBRARIES)

add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake) 

# Unix, linux:
if(UNIX)
   set_target_properties(
      ${LIB_NAME}
      PROPERTIES
      VERSION ${LINK_VERSION}
      SOVERSION ${LINK_SOVERSION}
      CLEAN_DIRECT_OUTPUT 1 )
   # TODO: test   
   if (APPLE)
      set_target_properties(
        ${LIB_NAME}
        PROPERTIES
        INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib")
   endif()
endif()

install( TARGETS ${LIB_NAME}
    EXPORT depends
    RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
    ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
    LIBRARY DESTINATION "${INSTALL_LIB_DIR}" )

# Define grouping of source files in PROJECT file (e.g. Visual Studio)
source_group("CMake Files" FILES CMakeLists.txt)
source_group("Header Files" FILES ${GEOTIFF_LIB_HEADERS})
source_group("Source Files" FILES ${GEOTIFF_LIB_SOURCES})


