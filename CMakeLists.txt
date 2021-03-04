cmake_minimum_required(VERSION 2.8.3)
project(ndt_matching)

find_package(PCL REQUIRED)

find_package(OpenMP)
if (OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif ()

find_package(Eigen3 QUIET)

if (NOT EIGEN3_FOUND)
    # Fallback to cmake_modules
    find_package(cmake_modules REQUIRED)
    find_package(Eigen REQUIRED)
    set(EIGEN3_INCLUDE_DIRS ${EIGEN_INCLUDE_DIRS})
    set(EIGEN3_LIBRARIES ${EIGEN_LIBRARIES})  # Not strictly necessary as Eigen is head only
    # Possibly map additional variables to the EIGEN3_ prefix.
else ()
    set(EIGEN3_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR})
endif ()

find_package(catkin REQUIRED COMPONENTS
        roscpp
        std_msgs
        nav_msgs
        tf
        pcl_ros
        sensor_msgs
        pcl_conversions
        velodyne_pointcloud
        jsk_rviz_plugins
        )

###################################
## catkin specific configuration ##
###################################
catkin_package(
        CATKIN_DEPENDS
        std_msgs
        velodyne_pointcloud
        jsk_rviz_plugins
        DEPENDS PCL
)

###########
## Build ##
###########

include_directories(
        "include"
        ${catkin_INCLUDE_DIRS}
        ${EIGEN3_INCLUDE_DIRS}
        ${PCL_INCLUDE_DIRS}
)

SET(CMAKE_CXX_FLAGS "-O2 -g -Wall ${CMAKE_CXX_FLAGS}")

set(
        srcs_ndt
        src/NormalDistributionsTransform.cpp
        src/Registration.cpp
        src/VoxelGrid.cpp
        src/Octree.cpp
)

set(
        incs_ndt
        include/ndt_cpu/debug.h
        include/ndt_cpu/NormalDistributionsTransform.h
        include/ndt_cpu/Registration.h
        include/ndt_cpu/SymmetricEigenSolver.h
        include/ndt_cpu/VoxelGrid.h
        include/ndt_cpu/Octree.h
)

add_definitions(-DEIGEN_DISABLE_UNALIGNED_ARRAY_ASSERT)

add_library(ndt_cpu SHARED ${incs_ndt} ${srcs_ndt})
        
        

set(
        srcs_omp
        src/ndt.cpp
)

set(
        incs_omp
        include/pcl_omp_registration/ndt.h
        include/pcl_omp_registration/registration.h
)

set(
        impl_incs
        include/pcl_omp_registration/impl/ndt.hpp
        include/pcl_omp_registration/impl/registration.hpp
)


add_library(
        pcl_omp_registration
        ${srcs_omp}
        ${incs_omp}
        ${impl_incs}
)

add_executable(ndt_matching nodes/ndt_matching/ndt_matching.cpp)
target_link_libraries(ndt_matching ${catkin_LIBRARIES} ndt_cpu pcl_omp_registration)
add_dependencies(ndt_matching ${catkin_EXPORTED_TARGETS})



if (NOT (PCL_VERSION VERSION_LESS "1.7.2"))
    set_target_properties(ndt_matching PROPERTIES COMPILE_DEFINITIONS "USE_PCL_OPENMP")
endif (NOT (PCL_VERSION VERSION_LESS "1.7.2"))



install(
        TARGETS
        ndt_matching
        ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
        LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
        RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
        )

install(DIRECTORY launch/
        DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch
        PATTERN ".svn" EXCLUDE)
