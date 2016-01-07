###############################################################################
#
# CMake configuration file to build libxtiff library
#
# Author: Mateusz Loskot <mateusz@loskot.net>
#
###############################################################################

ADD_LIBRARY(xtiff STATIC xtiff.c)

if(BUILD_SHARED_LIBS)
    set(LIB_NAME xtiff)
    add_library(${LIB_NAME} SHARED xtiff.c)    
    # Windows:
    if(WIN32 AND MSVC)
        set_target_properties(${LIB_NAME} PROPERTIES IMPORT_SUFFIX "_i.lib")
    endif(WIN32 AND MSVC)
else()    
    set(LIB_NAME xtiffstatic)
    add_library(${LIB_NAME} STATIC xtiff.c)
endif()  

set(EXPORT_TARGETS ${EXPORT_TARGETS} ${LIB_NAME} PAREN_SCOPE)
