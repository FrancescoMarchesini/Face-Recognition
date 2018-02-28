cmake_minimum_required(VERSION 2.8)
project(2_utility_ip_camera)

# setup tensorRT flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -ldl")	# -std=gnu++11

# Qt is used to load images (installed by ubuntu-desktop)
find_package(Qt4 REQUIRED)
include(${QT_USE_FILE})
add_definitions(${QT_DEFINITIONS})

# setup CUDA
find_package(CUDA)
set(
        CUDA_NVCC_FLAGS
        ${CUDA_NVCC_FLAGS};
    -O3
        -gencode arch=compute_53,code=sm_53
        -gencode arch=compute_62,code=sm_62
)

# setup project output paths
set(PROJECT_OUTPUT_DIR  ${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR})
set(PROJECT_INCLUDE_DIR ${PROJECT_OUTPUT_DIR}/include)

file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR})
file(MAKE_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)

message("-- system arch:  ${CMAKE_SYSTEM_PROCESSOR}")
message("-- output path:  ${PROJECT_OUTPUT_DIR}")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)

############################
find_package(PkgConfig)
pkg_check_modules(GST REQUIRED gstreamer-1.0>=1.4
                               gstreamer-sdp-1.0>=1.4
                               gstreamer-video-1.0>=1.4
                               gstreamer-app-1.0>=1.4)

############################

# build C/C++ interface
include_directories(${PROJECT_INCLUDE_DIR} ${GIE_PATH}/include)
include_directories(/usr/include/gstreamer-1.0 /usr/lib/aarch64-linux-gnu/gstreamer-1.0/include /usr/include/glib-2.0 /usr/include/libxml2 /usr/lib/aarch64-linux-gnu/glib-2.0/include/)

file(GLOB inferenceSources *.cpp *.cu util/*.cpp util/camera/*.cpp util/cuda/*.cu util/display/*.cpp)
file(GLOB inferenceIncludes *.h util/*.h util/camera/*.h util/cuda/*.h util/display/*.h)

message(" librerie neccessarie all'utilizzo di glib ${GST_LIBRARIES}")
cuda_add_library(myApp SHARED ${inferenceSources})
target_link_libraries(myApp nvcaffe_parser nvinfer Qt4::QtGui GL GLEW ${GST_LIBRARIES} gstreamer-1.0 gstapp-1.0 )		# gstreamer-0.10 gstbase-0.10 gstapp-0.10

# transfer all headers to the include directory
foreach(include ${inferenceIncludes})
        message("-- Copying ${include}")
        configure_file(${include} ${PROJECT_INCLUDE_DIR} COPYONLY)
endforeach()

add_subdirectory(App)
add_subdirectory(App_bin)

# install
foreach(include ${inferenceIncludes})
    install(FILES "${include}" DESTINATION include/jetson-inference)
endforeach()

# install the shared library
install(TARGETS myApp DESTINATION lib/jetson-inference EXPORT jetson-inferenceConfig)

# install the cmake project, for importing
install(EXPORT jetson-inferenceConfig DESTINATION share/jetson-inference/cmake)
