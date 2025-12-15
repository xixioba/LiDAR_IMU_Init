cmake_minimum_required(VERSION 3.5)
project(lidar_imu_init)

SET(CMAKE_BUILD_TYPE "Release")

ADD_COMPILE_OPTIONS(-std=c++17 )
ADD_COMPILE_OPTIONS(-std=c++17 )
set( CMAKE_CXX_FLAGS "-std=c++17 -O3" ) 

add_definitions(-DROOT_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/\")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fexceptions" )
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -pthread -std=c++0x -fexceptions")

# macOS + Conda Environment Support
if(APPLE AND DEFINED ENV{CONDA_PREFIX})
    message(STATUS "Detected macOS conda environment: $ENV{CONDA_PREFIX}")
    
    # Add conda include path FIRST with highest priority
    set(CMAKE_CXX_FLAGS "-isystem $ENV{CONDA_PREFIX}/include/eigen3 -isystem $ENV{CONDA_PREFIX}/include ${CMAKE_CXX_FLAGS}")
    set(CMAKE_C_FLAGS "-isystem $ENV{CONDA_PREFIX}/include ${CMAKE_C_FLAGS}")
    
    # Explicitly tell compiler to skip /opt/homebrew/include to avoid Eigen conflicts
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-command-line-argument")
    
    # Also add to include directories for better IDE support
    include_directories(BEFORE SYSTEM $ENV{CONDA_PREFIX}/include/eigen3)
    include_directories(BEFORE SYSTEM $ENV{CONDA_PREFIX}/include)
endif()

message("Current CPU architecture: ${CMAKE_SYSTEM_PROCESSOR}")
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)" )
  include(ProcessorCount)
  ProcessorCount(N)
  message("Processer number:  ${N}")
  if(N GREATER 4)
    add_definitions(-DMP_EN)
    add_definitions(-DMP_PROC_NUM=3)
    message("core for MP: 3")
  elseif(N GREATER 3)
    add_definitions(-DMP_EN)
    add_definitions(-DMP_PROC_NUM=2)
    message("core for MP: 2")
  else()
    add_definitions(-DMP_PROC_NUM=1)
  endif()
else()
  add_definitions(-DMP_PROC_NUM=1)
endif()

find_package(OpenMP QUIET)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}   ${OpenMP_C_FLAGS}")

find_package(PythonLibs REQUIRED)
find_path(MATPLOTLIB_CPP_INCLUDE_DIRS "matplotlibcpp.h")

execute_process(
  COMMAND ${PYTHON_EXECUTABLE} -c "import numpy; print(numpy.get_include())"
  OUTPUT_VARIABLE NUMPY_INCLUDE_DIR
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
message(STATUS "Found NumPy include dir: ${NUMPY_INCLUDE_DIR}")

find_package(catkin REQUIRED COMPONENTS
  geometry_msgs
  nav_msgs
  sensor_msgs
  roscpp
  rospy
  std_msgs
  pcl_ros
  tf
  # livox_ros_driver
  message_generation
  eigen_conversions
)

find_package(Eigen3 REQUIRED)
find_package(PCL 1.8 REQUIRED)
find_package(Ceres REQUIRED)

message(Eigen: ${EIGEN3_INCLUDE_DIR})

add_message_files(
  FILES
  Pose6D.msg
  States.msg
  CustomMsg.msg
  CustomPoint.msg
)

include_directories(
     ../../devel/include
	${catkin_INCLUDE_DIRS} 
  ${EIGEN3_INCLUDE_DIR}
  ${PCL_INCLUDE_DIRS}
  ${PYTHON_INCLUDE_DIRS}
  ${CERES_INCLUDE_DIRS}
  ${NUMPY_INCLUDE_DIR}  # Add numpy include path
  include)

generate_messages(
 DEPENDENCIES
 geometry_msgs
)

catkin_package(
  CATKIN_DEPENDS geometry_msgs nav_msgs roscpp rospy std_msgs message_runtime
  DEPENDS EIGEN3 PCL
  INCLUDE_DIRS
)

add_executable(li_init
        src/laserMapping.cpp
        include/ikd-Tree/ikd_Tree.cpp
        include/LI_init/LI_init.cpp
        src/preprocess.cpp)
add_dependencies(li_init ${${PROJECT_NAME}_EXPORTED_TARGETS} ${catkin_EXPORTED_TARGETS})

target_link_libraries(li_init ${catkin_LIBRARIES} ${PCL_LIBRARIES} ${PYTHON_LIBRARIES} ${CERES_LIBRARIES})
target_include_directories(li_init PRIVATE ${PYTHON_INCLUDE_DIRS})




