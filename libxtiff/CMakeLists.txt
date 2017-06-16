###############################################################################
#
# CMake configuration file to build libxtiff library
#
# Author: Mateusz Loskot <mateusz@loskot.net>
#
###############################################################################

if(NOT BUILD_SHARED_LIBS)
    set(XLIB_NAME xtiff)
    add_library(${XLIB_NAME} STATIC xtiff.c)

    if(WITH_TIFF_EXTERNAL)
        add_dependencies(${XLIB_NAME} TIFF_EP)
    endif()
    if(WITH_PROJ4_EXTERNAL)
        add_dependencies(${XLIB_NAME} PROJ4_EP)
    endif()
    if(WITH_ZLIB_EXTERNAL)
        add_dependencies(${XLIB_NAME} ZLIB_EP)
    endif()
    if(WITH_JPEG_EXTERNAL)
        add_dependencies(${XLIB_NAME} JPEG_EP)
    endif()

    set(EXPORT_TARGETS ${EXPORT_TARGETS} ${XLIB_NAME} PARENT_SCOPE)
    set(TARGET_LINK_LIB ${TARGET_LINK_LIB} PARENT_SCOPE)
endif()
