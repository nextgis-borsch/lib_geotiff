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

if(WIN32)
    file(GLOB_RECURSE IMPORTED_WIN_DLLS ${CMAKE_BINARY_DIR}/third-party/*.dll)
    set(IMPORTED_WIN_DLLS ${IMPORTED_WIN_DLLS} $<TARGET_FILE:geotiff>)
    foreach(IMPORTED_WIN_DLL ${IMPORTED_WIN_DLLS})
        add_custom_command(TARGET geotifcp
                            POST_BUILD
                            COMMAND ${CMAKE_COMMAND} -E copy ${IMPORTED_WIN_DLL} $<TARGET_FILE_DIR:geotifcp>
                            COMMENT "Copy ${IMPORTED_WIN_DLL} to geotifcp target directory"
                        )
    endforeach()
endif()
