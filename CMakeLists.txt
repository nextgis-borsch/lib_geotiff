###############################################################################
#
# CMake main configuration file to build GeoTIFF library and utilities.
#
# Author: Mateusz Loskot <mateusz@loskot.net>
# Author: Dmitry Baryshnikov <polimax@mail.ru>
# Copyright (C), 2016 NextGIS <info@nextgis.com>
#
###############################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(geotiff)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
  set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
  set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
  set(INSTALL_INC_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Headers CACHE INTERNAL "Installation directory for headers" FORCE)
  set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
  set(SKIP_INSTALL_HEADERS ON)
  set(SKIP_INSTALL_EXECUTABLES ON)
  set(SKIP_INSTALL_FILES ON)
  set(SKIP_INSTALL_DOCS ON)
  set(SKIP_INSTALL_EXPORT ON)
  set(CMAKE_MACOSX_RPATH ON)
else()
    include(GNUInstallDirs)

    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_SHARE_DIR ${CMAKE_INSTALL_DATADIR} CACHE INTERNAL "Shared files directory" FORCE)
    set(INSTALL_DOC_DIR ${CMAKE_INSTALL_DOCDIR} CACHE INTERNAL "Doc files directory" FORCE)
    set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
endif()

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

report_version(${PROJECT_NAME} "${VERSION} (${LINK_VERSION})")

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${PROJECT_VERSION_MAJOR}")
endif()

###############################################################################
# General build settings

set(GEOTIFF_BUILD_PEDANTIC FALSE CACHE BOOL "Choose compilation in pedantic or relaxed mode")
if(CMAKE_BUILD_TYPE MATCHES Debug)
    set(GEOTIFF_BUILD_PEDANTIC TRUE)
endif()

if(CMAKE_MAJOR_VERSION GREATER 2)
   cmake_policy(SET CMP0022 OLD) # interface link libraries
   cmake_policy(SET CMP0042 OLD) # osx rpath
endif()

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
    set(_COMPILE_FLAGS "-fPIC -Wall -Wno-long-long -Wno-implicit-function-declaration -Wno-int-conversion")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${_COMPILE_FLAGS} -std=c99")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${_COMPILE_FLAGS} -std=c++98")
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

configure_file(${CMAKE_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake IMMEDIATE @ONLY)

include(FindAnyProject)

###############################################################################
# User-defined build settings

set(GEOTIFF_CSV_NAMES area.csv codes.csv datum.csv gcs.csv pcs.csv)
find_exthost_path(GEOTIFF_CSV_DATA_DIR NAMES ${GEOTIFF_CSV_NAMES}
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

# TIFF support - required, default=ON
find_anyproject(TIFF REQUIRED CMAKE_ARGS
    -DSKIP_BUILD_DOCS=ON
)
if(TIFF_FOUND)
    add_definitions(-DHAVE_TIFF=1)
endif(TIFF_FOUND)

# PROJ.4 support - optional, default=ON
find_anyproject(PROJ4 DEFAULT ON)
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

include_directories(${PROJECT_SOURCE_DIR})
include_directories(${PROJECT_SOURCE_DIR}/libxtiff)
include_directories(${CMAKE_CURRENT_BINARY_DIR})


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
if(NOT SKIP_INSTALL_DOCS AND NOT SKIP_INSTALL_ALL)
install(FILES
    AUTHORS ChangeLog COPYING INSTALL LICENSE README README_BIN README.WIN
    DESTINATION "${INSTALL_DOC_DIR}")
endif()

# Install CSV data files
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL)
    install(FILES ${GEOTIFF_CSV_DATA} DESTINATION ${INSTALL_SHARE_DIR}/epsg_csv)
endif()
# Install header files for development distribution
if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
    install(FILES ${GEOTIFF_LIB_HEADERS} DESTINATION ${INSTALL_INC_DIR})
endif()

set(EXPORT_TARGETS)

###############################################################################
# Build libxtiff library

# add_subdirectory(libxtiff)

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
    if(INCODE_EPSG_FROM_CSV)
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

if (MSVC OR CMAKE_CONFIGURATION_TYPES)
  # For multi-config systems and for Visual Studio, the debug versions
  # of the libraries have a _d suffix.
  set (CMAKE_DEBUG_POSTFIX _d)
endif ()

set(LIB_NAME ${PROJECT_NAME})
if(BUILD_SHARED_LIBS)
    add_library(${LIB_NAME} SHARED ${GEOTIFF_LIB_SOURCES} ${GEOTIFF_LIB_CSV_SOURCES} ${XTIFF_SOURCES})
    # Windows:
    if(WIN32 AND MSVC)
        set_target_properties(${LIB_NAME} PROPERTIES IMPORT_SUFFIX "_i.lib")
    endif(WIN32 AND MSVC)
elseif(OSX_FRAMEWORK)
    add_library(${LIB_NAME} SHARED ${GEOTIFF_LIB_SOURCES} ${GEOTIFF_LIB_CSV_SOURCES} ${XTIFF_SOURCES} ${GEOTIFF_LIB_HEADERS} ${GEOTIFF_CSV_DATA})
    set_property(SOURCE ${GEOTIFF_CSV_DATA} PROPERTY MACOSX_PACKAGE_LOCATION "Resources/epsg_csv")
    set_target_properties(${LIB_NAME} PROPERTIES
      FRAMEWORK TRUE
      FRAMEWORK_VERSION "${FRAMEWORK_VERSION}"
      MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
      MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
      MACOSX_FRAMEWORK_IDENTIFIER org.osgeo.libgeotiff
      XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
      PUBLIC_HEADER "${GEOTIFF_LIB_HEADERS}"
    )
else()
    add_library(${LIB_NAME} STATIC ${GEOTIFF_LIB_SOURCES} ${GEOTIFF_LIB_CSV_SOURCES} ${XTIFF_SOURCES})
endif()

target_link_extlibraries(${LIB_NAME})

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

# Unix, linux:
if(UNIX)
   set_target_properties(${LIB_NAME}
      PROPERTIES
      VERSION ${LINK_VERSION}
      SOVERSION ${LINK_SOVERSION}
      CLEAN_DIRECT_OUTPUT 1 )
endif()

install( TARGETS ${LIB_NAME}
    EXPORT ${PACKAGE_UPPER_NAME}Targets
    RUNTIME DESTINATION ${INSTALL_BIN_DIR}
    LIBRARY DESTINATION ${INSTALL_LIB_DIR}
    ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
    INCLUDES DESTINATION ${INSTALL_INC_DIR}
    FRAMEWORK DESTINATION ${INSTALL_LIB_DIR})

# Define grouping of source files in PROJECT file (e.g. Visual Studio)
source_group("CMake Files" FILES CMakeLists.txt)
source_group("Header Files" FILES ${GEOTIFF_LIB_HEADERS})
source_group("Source Files" FILES ${GEOTIFF_LIB_SOURCES})


if(WITH_UTILITIES)
    find_anyproject(JBIG REQUIRED)
    find_anyproject(LibLZMA REQUIRED)
    set(GEOTIFF_LIBRARY_TARGET ${LIB_NAME} ${TARGET_LINK_LIB})
    add_subdirectory(bin)
endif()

enable_testing()
if(BUILD_TESTING)
    if(NOT WITH_UTILITIES)
        message(FATAL_ERROR "Build utilities required. Set WITH_UTILITIES to TRUE.")
    endif()
    # Just test if applications are executing
    add_test(makegeo bin/makegeo)
    add_test(listgeo bin/listgeo newgeo.tif)

endif()

# uninstall
add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# Export package ===============================================================

# Add path to includes to build-tree export
target_include_directories(${LIB_NAME} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libxtiff>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)

# Add all targets to the build-tree export set
export(TARGETS ${LIB_NAME}
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the <Package>Config.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
