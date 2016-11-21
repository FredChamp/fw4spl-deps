cmake_minimum_required(VERSION 3.0)

project(pythonBuilder)

include(ExternalProject)

set(CACHED_URL http://www.python.org/ftp/python/2.7.12/Python-2.7.12.tgz)

if(WIN32)
    set(PYTHON_PATCH_DIR ${CMAKE_CURRENT_SOURCE_DIR}/patch)
    set(PYTHON_CMAKE_ARGS 
        -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DENABLE_CURSES:BOOL=OFF
        -DENABLE_CURSES_PANEL:BOOL=OFF
        -DENABLE_CRYPT:BOOL=OFF
        -DENABLE_MMAP:BOOL=OFF
        -DENABLE_IO:BOOL=OFF
        -DENABLE_MATH:BOOL=OFF
        -DENABLE_PARSER:BOOL=OFF
        -DENABLE_SHARED:BOOL=ON
        -DENABLE_STATIC:BOOL=OFF
    )
    ExternalProject_Add(
        python
        URL ${CACHED_URL}
        URL_HASH SHA256=74d70b914da4487aa1d97222b29e9554d042f825f26cb2b93abd20fdda56b557
        DOWNLOAD_DIR ${ARCHIVE_DIR}
        DEPENDS zlib expat
        PATCH_COMMAND ${CMAKE_COMMAND} -E  copy_directory "${PYTHON_PATCH_DIR}/" "<SOURCE_DIR>/"
        INSTALL_DIR ${CMAKE_INSTALL_PREFIX}
        CMAKE_ARGS ${PYTHON_CMAKE_ARGS}
    )
else(WIN32)

    if(NOT APPLE)
        if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
            set(SETENV export CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER} CFLAGS=-Wno-unused-value &&)  
        endif()
    endif()

    set(CONFIGURE_CMD <SOURCE_DIR>/configure
                      --enable-shared
                      #--disable-static
                      --prefix=${CMAKE_INSTALL_PREFIX}
                      #--with-system-expat
                      --enable-unicode=ucs4
                      #--with-expat-prefix=${CMAKE_INSTALL_PREFIX}
                      #--with-expat-include-prefix=${CMAKE_INSTALL_PREFIX}/include 
                      #--with-expat-libs-prefix=${CMAKE_INSTALL_PREFIX}/lib
    )
    
    if(${IS_DEBUG})
        list(APPEND CONFIGURE_CMD --with-debug --with-pydebug)
    else(${IS_DEBUG})
        #list(APPEND CONFIGURE_CMD --without-debug)
    endif(${IS_DEBUG})
    ExternalProject_Add(
        python
        URL ${CACHED_URL}
        URL_HASH SHA256=3cb522d17463dfa69a155ab18cffa399b358c966c0363d6c8b5b3bf1384da4b6
        DOWNLOAD_DIR ${ARCHIVE_DIR}
        BUILD_IN_SOURCE 1
        DEPENDS expat
        CONFIGURE_COMMAND ${SETENV} ${CONFIGURE_CMD}
        BUILD_COMMAND ${MAKE} -f <SOURCE_DIR>/Makefile -j${NUMBER_OF_PARALLEL_BUILD}
        INSTALL_COMMAND ${MAKE} -f <SOURCE_DIR>/Makefile DESTDIR=${INSTALL_PREFIX_python} install
    )
    ExternalProject_Add_Step(python COPY_FILES
        COMMAND ${CMAKE_COMMAND} -D SRC:PATH=${INSTALL_PREFIX_python} -D DST:PATH=${CMAKE_INSTALL_PREFIX} -P ${CMAKE_SOURCE_DIR}/Install.txt
        DEPENDEES install
    )
endif(WIN32)


ExternalProject_Add_Step(python CopyConfigFileToInstall
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/cmake/findBinpkgs/fw-python.cmake ${CMAKE_INSTALL_PREFIX}/fw-python.cmake
    COMMENT "Install configuration file"
    DEPENDEES install
)
