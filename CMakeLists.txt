cmake_minimum_required(VERSION 3.8)

project(ProgressiveX LANGUAGES CXX)

# indicate if OPENMP should be enabled
option(USE_OPENMP "Use OPENMP" ON)

# indicate if the sample project should be included
option(CREATE_SAMPLE_PROJECT "Create the Sample Project" OFF)

# indicate if GFLags should be enabled
option(USE_GFLAGS "Enable Google Flags." ON)

# indicate if GLog should be enabled
option(USE_GLOG "Enable Google Log." ON)

SET (CMAKE_BUILD_TYPE "RELEASE")

IF(CMAKE_COMPILER_IS_GNUCXX)
	#SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-strict-aliasing")
	#SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wextra -Wno-write-strings")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-deprecated -ansi")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -ftree-vectorize -funroll-loops")
ENDIF(CMAKE_COMPILER_IS_GNUCXX)

# ==============================================================================
# Check C++17 support
# ==============================================================================
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ==============================================================================
# Find OpenCV
# ==============================================================================
find_package(OpenCV)
if (OPENCV_FOUND)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenCV_CXX_FLAGS}")
  	add_definitions(-DUSE_OPENMP)
	set(TRGT_LNK_LBS_ADDITIONAL OpenMP::OpenMP_CXX)
endif (OPENCV_FOUND)

# ==============================================================================
# Find Eigen
# ==============================================================================
find_package(Eigen3 REQUIRED)
set(EIGEN_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR})

# ==============================================================================
# Find GFlags
# ==============================================================================
find_package(gflags REQUIRED)
if(DEFINED gflags_VERSION_STRING OR DEFINED GFLAGS_VERSION_STRING)
        # different cmake variable with lower or upper case exist
        set(GFLAGS_INCLUDE_DIR ${GFLAGS_INCLUDE_DIR})
        #set(GFLAGS_INCLUDE_DIR ${GFLAGS_INCLUDE_DIR} ${gflags_INCLUDE_DIR})
        set(GFLAGS_LIBRARIES ${GFLAGS_LIBRARIES})
        #set(GFLAGS_LIBRARIES ${GFLAGS_LIBRARIES} ${gflags_LIBRARIES})
  message( "Found Gflags ${GFLAGS_VERSION_STRING}" )
  message( STATUS "> GFLAGS_INCLUDE_DIR:   ${GFLAGS_INCLUDE_DIR}" )
  message( STATUS "> GFLAGS_LIBRARIES:   ${GFLAGS_LIBRARIES}" )
endif()

# ==============================================================================
# Find GLOG
# ==============================================================================
#include(FindPackageHandleStandardArgs)
#
#set(GLOG_ROOT_DIR "" CACHE PATH "Folder contains Google glog")
# 
#find_path(GLOG_INCLUDE_DIR glog/logging.h
#    PATHS
#        ${GLOG_ROOT_DIR}
#    PATH_SUFFIXES
#        src)
# 
#find_library(GLOG_LIBRARY glog libglog
#    PATHS
#        ${GLOG_ROOT_DIR}
#    PATH_SUFFIXES
#        .libs
#        lib
#        lib64)
# 
#find_package_handle_standard_args(GLOG DEFAULT_MSG
#    GLOG_INCLUDE_DIR GLOG_LIBRARY)
# 
#if(GLOG_FOUND)
#    set(GLOG_INCLUDE_DIRS ${GLOG_INCLUDE_DIR})
#    set(GLOG_LIBRARIES ${GLOG_LIBRARY})
#endif()
#

#find_package(glog REQUIRED)
#if(NOT glog_FOUND)
#	message(FATAL_ERROR "Glog not found.")
#endif()

# FindPkgConfig workflow
find_package(PkgConfig REQUIRED)
pkg_check_modules(glog REQUIRED libglog)

# ==============================================================================
# Find OpenMP
# ==============================================================================
find_package(OpenMP)
if (USE_OPENMP)
	if(NOT OPENMP_FOUND)
		message(FATAL_ERROR "OPENMP not found.")
	endif()
	add_definitions(-DUSE_OPENMP)
	set(TRGT_LNK_LBS_ADDITIONAL OpenMP::OpenMP_CXX)
endif (USE_OPENMP)

# ==============================================================================
# Includes
# ==============================================================================
SET(SOURCE_DIR "src/pyprogressivex")

include_directories (
	${PROJECT_SOURCE_DIR}/graph-cut-ransac/src/pygcransac/include
	${glog_INCLUDE_DIRS} #${GLOG_INCLUDE_DIR}
)

# ==============================================================================
# Structure: Graph-Cut RANSAC Library
# ==============================================================================
# Set header files for the library
file(GLOB_RECURSE HDRS_GraphCutRANSAC
	"graph-cut-ransac/src/pygcransac/include/*.h"
) 

# Set source files to be added to the library
file(GLOB_RECURSE SRCS_GraphCutRANSAC 
	"graph-cut-ransac/src/pygcransac/include/*.cpp"
)

add_library(GraphCutRANSAC SHARED
	${HDRS_GraphCutRANSAC}
	${SRCS_GraphCutRANSAC}
)

target_link_libraries(GraphCutRANSAC
	${OpenCV_LIBS}
	Eigen3::Eigen)

# ==============================================================================
# Structure: Progressive-X Library
# ==============================================================================

# Tell cmake that headers are in alse in source_dir
include_directories(${SOURCE_DIR}/include)

# Set header files for the library
file(GLOB_RECURSE HDRS_PROGRESSIVEX
	"${SOURCE_DIR}/include/*.h"
) 

# Set source files to be added to the library
file(GLOB_RECURSE SRCS_PROGRESSIVEX 
	"${SOURCE_DIR}/include/*.cpp"
)

# ==============================================================================
# Structure: Python binding
# ==============================================================================
# Generate python module
if(0)
add_subdirectory(lib/pybind11)

pybind11_add_module(pyprogressivex ${SOURCE_DIR}/src/bindings.cpp ${SOURCE_DIR}/src/progressivex_python.cpp  ${HDRS_PROGRESSIVEX} ${SRCS_PROGRESSIVEX} )
target_compile_options(pyprogressivex PRIVATE -O3 -std=c++17)
target_link_libraries(pyprogressivex PRIVATE  
	${OpenCV_LIBS} 
	Eigen3::Eigen 
	${glog_LIBRARIES} #glog::glog
	gflags
	GraphCutRANSAC)
endif(0)

add_library(${PROJECT_NAME} STATIC
	${HDRS_PROGRESSIVEX}
	${SRCS_PROGRESSIVEX}
)

target_link_libraries (${PROJECT_NAME}
	${OpenCV_LIBS}
	Eigen3::Eigen
	${glog_LIBRARIES} #glog::glog
	gflags
	GraphCutRANSAC)
	
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
	
# ==============================================================================
# Structure: Applications
# ==============================================================================
if (CREATE_SAMPLE_PROJECT)
	set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/build)

	add_executable(SampleProject
		examples/cpp_example.cpp)
		
	target_link_libraries(SampleProject PUBLIC
		${OpenCV_LIBS}
		${PROJECT_NAME}
		Eigen3::Eigen
		gflags
		${glog_LIBRARIES} #glog::glog
		GraphCutRANSAC
	)

	add_dependencies(SampleProject ${PROJECT_NAME})
endif (CREATE_SAMPLE_PROJECT)
