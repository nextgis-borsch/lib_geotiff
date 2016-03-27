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
    
    set(EXPORT_TARGETS ${EXPORT_TARGETS} ${XLIB_NAME} PARENT_SCOPE)
    set(TARGET_LINK_LIB ${TARGET_LINK_LIB} PARENT_SCOPE)
endif()
