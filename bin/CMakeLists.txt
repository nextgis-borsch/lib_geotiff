###############################################################################
#
# CMake configuration file to build GeoTIFF utility programs
#
# Author: Mateusz Loskot <mateusz@loskot.net>
#
###############################################################################

INCLUDE_DIRECTORIES(
    .
    ${CMAKE_SOURCE_DIR})

IF(WIN32 AND MSVC)
    SET(GETOPT_SOURCE getopt.c)
ENDIF()
###############################################################################
# Collect programs to build

SET(GEOTIFF_UTILITIES makegeo listgeo applygeo)

MESSAGE(STATUS "Adding GeoTIFF utilities to build")

FOREACH(utility ${GEOTIFF_UTILITIES})
    ADD_EXECUTABLE(${utility} ${utility}.c ${GETOPT_SOURCE})
    TARGET_LINK_LIBRARIES(${utility} ${GEOTIFF_LIBRARY_TARGET})
    if(OSX_FRAMEWORK)
        set_target_properties(${utility} PROPERTIES INSTALL_RPATH "@executable_path/../../Library/Frameworks")
    endif()
ENDFOREACH()

ADD_EXECUTABLE(geotifcp geotifcp.c ${GETOPT_SOURCE})
TARGET_LINK_LIBRARIES(geotifcp
    ${GEOTIFF_LIBRARY_TARGET}
    ${JPEG_LIBRARIES}
    ${ZLIB_LIBRARIES})
if(OSX_FRAMEWORK)
    set_target_properties(geotifcp PROPERTIES INSTALL_RPATH "@executable_path/../../Library/Frameworks")
endif()

SET(GEOTIFF_UTILITIES ${GEOTIFF_UTILITIES} geotifcp )

MESSAGE(STATUS "Adding GeoTIFF utilities to build - done: ${GEOTIFF_UTILITIES}")

###############################################################################
# Targets installation

INSTALL(TARGETS ${GEOTIFF_UTILITIES}
    RUNTIME DESTINATION ${INSTALL_BIN_DIR}
    LIBRARY DESTINATION ${INSTALL_LIB_DIR}
    ARCHIVE DESTINATION ${INSTALL_LIB_DIR})

# Copy dependencies to bin dir
if(WIN32)
    add_custom_command(TARGET geotifcp
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -DDST_PATH="$<TARGET_FILE_DIR:geotifcp>" -DSRC_PATH="${PROJECT_BINARY_DIR}" -P ${PROJECT_SOURCE_DIR}/cmake/test_dependency.cmake
        COMMENT "Copy DLLs"
    )
endif()