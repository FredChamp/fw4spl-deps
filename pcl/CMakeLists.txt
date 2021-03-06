cmake_minimum_required(VERSION 3.0)

project(pclBuilder)

include(ExternalProject)

set(CACHED_URL https://github.com/PointCloudLibrary/pcl/archive/pcl-1.8.0.tar.gz)

include( ${CMAKE_SOURCE_DIR}/cmake/findBinpkgs/fw-boost.cmake )

if(WIN32)
    set(OPENNI2_LIBRARY "OpenNI2.lib")
elseif(APPLE)
    set(OPENNI2_LIBRARY "libOpenNI2.dylib")
else()
    set(OPENNI2_LIBRARY "libOpenNI2.so")
endif()

set(PCL_CMAKE_ARGS      ${COMMON_CMAKE_ARGS}
                        -DBUILD_apps:BOOL=OFF
                        -DBUILD_OPENNI:BOOL=OFF
                        -DBUILD_OPENNI2:BOOL=ON
                        -DBUILD_DAVIDSDK:BOOL=OFF
                        -DBUILD_ENSENSO:BOOL=OFF
                        -DBUILD_examples:BOOL=OFF
                        -DBUILD_common:BOOL=ON
                        -DBUILD_2d:BOOL=ON
                        -DBUILD_features:BOOL=ON
                        -DBUILD_filters:BOOL=ON
                        -DBUILD_geometry:BOOL=ON
                        -DBUILD_io:BOOL=ON
                        -DBUILD_kdtree:BOOL=ON
                        -DBUILD_octree:BOOL=ON
                        -DBUILD_sample_consensus:BOOL=ON
                        -DBUILD_search:BOOL=ON
                        -DBUILD_tools:BOOL=OFF
                        -DCMAKE_C_COMPILER:PATH=${CMAKE_C_COMPILER}
                        -DCMAKE_CXX_COMPILER:PATH=${CMAKE_CXX_COMPILER}
                        -DBoost_COMPILER:STRING=${Boost_COMPILER}
                        -DBOOST_ROOT:PATH=${CMAKE_INSTALL_PREFIX}
                        -DBOOST_INCLUDEDIR:PATH=${CMAKE_INSTALL_PREFIX}/include/boost-${BOOST_VERSION}
                        -DBoost_USE_DEBUG_PYTHON:BOOL=ON
                        -DBoost_USE_MULTITHREADED:BOOL=ON
                        -DBoost_USE_STATIC_LIBS:BOOL=OFF
                        -DPCL_BUILD_WITH_BOOST_DYNAMIC_LINKING_WIN32:BOOL=ON
                        -DFLANN_INCLUDE_DIR:PATH=${CMAKE_INSTALL_PREFIX}/include/flann
                        -DVTK_DIR:PATH=${CMAKE_INSTALL_PREFIX}/lib/
                        -DPCL_SHARED_LIBS:BOOL=ON
                        -DZLIB_INCLUDE_DIR:PATH=${CMAKE_INSTALL_PREFIX}/include
                        -DWITH_PCAP:BOOL=OFF
                        -DCUDA_TOOLKIT_ROOT_DIR:BOOL=${CUDA_TOOLKIT_ROOT_DIR}
                        -DWITH_RSSDK:BOOL=OFF
)

set(PCL_DEPENDS
    zlib
    boost
    eigen
    flann
)

if(NOT ANDROID)
    list(APPEND PCL_DEPENDS
        qt
        vtk
        openni
    )

    list(APPEND PCL_CMAKE_ARGS
        -DBUILD_TESTS:BOOL=OFF
        -DBUILD_WITH_DEBUG_INFO:BOOL=OFF
        -DOPENNI2_INCLUDE_DIRS:PATH=${CMAKE_INSTALL_PREFIX}/include/openni2
        -DOPENNI2_LIBRARY:PATH=${CMAKE_INSTALL_PREFIX}/lib/openni2/${OPENNI2_LIBRARY}
        -DQT_QMAKE_EXECUTABLE:PATH=${CMAKE_INSTALL_PREFIX}/bin/qmake
    )
else()
    list(APPEND PCL_CMAKE_ARGS
        -DBUILD_visualization:BOOL=OFF
        -DBUILD_global_tests:BOOL=OFF
        -DBUILD_people:BOOL=OFF
        -DWITH_OPENGL:BOOL=OFF
        -DHAVE_POSIX_MEMALIGN_EXITCODE:BOOL=OFF
    )
endif()

#OpenMP
find_package(OpenMP)
if(OPENMP_FOUND)
    list(APPEND PCL_CMAKE_ARGS -DENABLE_OPENMP=ON)
endif()

#CUDA
find_package(CUDA)
set(CUDA_HOST_COMPILER_SUPPORT TRUE)
if(MSVC AND (NOT (MSVC_VERSION LESS 1900)))
    # Microsoft Visual Studio 2015 Update >= 2 is not supported yet!
    set(CUDA_HOST_COMPILER_SUPPORT FALSE)
endif()
if(CUDA_FOUND AND CUDA_HOST_COMPILER_SUPPORT)
    #Enable GPU options
    list(APPEND PCL_CMAKE_ARGS -DBUILD_CUDA:BOOL=ON
                               -DBUILD_GPU:BOOL=ON
                               -DBUILD_gpu_kinfu:BOOL=ON
                               -DBUILD_gpu_kinfu_large_scale:BOOL=ON
                               -DBUILD_visualization:BOOL=ON
                               -DBUILD_surface:BOOL=ON
    )
    if (APPLE)
        list(APPEND PCL_CMAKE_ARGS -DBUILD_gpu_features:BOOL=OFF)
    endif()
    set(PCL_BUILD_COMMAND ${CMAKE_MAKE_PROGRAM})
    #Fix fatal error C1041
    if(WIN32)
        if(${CMAKE_MAKE_PROGRAM} STREQUAL "jom")
            set(PCL_TARGETS pcl_common pcl_octree pcl_io pcl_io_ply pcl_kdtree pcl_search pcl_sample_consensus pcl_filters
                            pcl_2d pcl_features pcl_ml pcl_segmentation pcl_visualization pcl_surface pcl_registration
                            pcl_keypoints pcl_tracking pcl_recognition pcl_stereo pcl_cuda_features pcl_cuda_segmentation
                            pcl_cuda_sample_consensus pcl_outofcore pcl_people)
            set(PCL_GPU_TARGETS pcl_gpu_containers pcl_gpu_utils pcl_gpu_octree pcl_gpu_features pcl_gpu_kinfu pcl_gpu_kinfu_large_scale pcl_gpu_segmentation)
            set(PCL_BUILD_COMMAND ${CMAKE_MAKE_PROGRAM} ${PCL_TARGETS}
                COMMAND ${CMAKE_MAKE_PROGRAM} -j1 ${PCL_GPU_TARGETS})
        endif()
    else()
        set(PCL_BUILD_COMMAND ${CMAKE_MAKE_PROGRAM} -j${NUMBER_OF_PARALLEL_BUILD})
        # Clang >= 3.8 is not supported by CUDA 7.5
        if(UNIX AND ${CMAKE_C_COMPILER_ID} STREQUAL "Clang" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 3.7)
            list(APPEND PCL_CMAKE_ARGS -DCUDA_HOST_COMPILER:PATH=/usr/bin/gcc
                                       -DCUDA_PROPAGATE_HOST_FLAGS:BOOL=OFF
            )
        endif()
    endif()
else()
    message("warning, no acceptable version of CUDA or supported compiler found")
    list(APPEND PCL_CMAKE_ARGS -DBUILD_tracking:BOOL=OFF
                               -DBUILD_segmentation:BOOL=OFF
                               -DBUILD_simulation:BOOL=OFF
                               -DBUILD_stereo:BOOL=OFF
                               -DBUILD_surface_on_nurbs:BOOL=OFF
                               -DBUILD_outofcore:BOOL=OFF
                               -DBUILD_recognition:BOOL=OFF
                               -DBUILD_registration:BOOL=OFF
                               -DBUILD_keypoints:BOOL=OFF
                               -DBUILD_ml:BOOL=OFF
    )
endif()

set(PCL_PATCH_DIR ${CMAKE_CURRENT_SOURCE_DIR}/patch)
set(PCL_PATCH_CMD "${PATCH_EXECUTABLE}" -p1 -i "${PCL_PATCH_DIR}/kinfu.diff" -d "<SOURCE_DIR>")
if(ANDROID)
    list (APPEND PCL_PATCH_CMD && "${PATCH_EXECUTABLE}" -p1 -i ${PCL_PATCH_DIR}/pcl_targets.diff -d <SOURCE_DIR>)
endif()
if(WIN32)
    list (APPEND PCL_PATCH_CMD && "${PATCH_EXECUTABLE}" -p1 -i ${PCL_PATCH_DIR}/win.diff -d <SOURCE_DIR>)
endif()

ExternalProject_Add(
    pcl
    URL ${CACHED_URL}
    URL_HASH SHA256=9e54b0c1b59a67a386b9b0f4acb2d764272ff9a0377b825c4ed5eedf46ebfcf4
    DOWNLOAD_DIR ${ARCHIVE_DIR}
    DEPENDS ${PCL_DEPENDS}
    PATCH_COMMAND ${PCL_PATCH_CMD}
    INSTALL_DIR ${CMAKE_INSTALL_PREFIX}
    CMAKE_ARGS ${PCL_CMAKE_ARGS}
    BUILD_COMMAND ${PCL_BUILD_COMMAND}
)
