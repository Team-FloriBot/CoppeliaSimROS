cmake_minimum_required(VERSION 3.16)
project(sim_ros_interface)

set(CMAKE_CXX_STANDARD 11)

set(PKG_DEPS
    roscpp
    rosconsole
    cv_bridge
    image_transport
    tf
    roslib
    actionlib_msgs
    control_msgs
    diagnostic_msgs
    geometry_msgs
    map_msgs
    nav_msgs
    sensor_msgs
    shape_msgs
    std_msgs
    tf2_geometry_msgs
    tf2_msgs
    trajectory_msgs
    visualization_msgs
    base)

find_package(catkin REQUIRED COMPONENTS ${PKG_DEPS})

if(NOT LIBPLUGIN_DIR)
    if(DEFINED ENV{COPPELIASIM_ROOT_DIR})
        set(LIBPLUGIN_DIR $ENV{COPPELIASIM_ROOT_DIR}/programming/libPlugin)
    else()
        message(FATAL_ERROR "Environment variable COPPELIASIM_ROOT_DIR is not set")
    endif()
endif()
set(CMAKE_MODULE_PATH
    ${CMAKE_MODULE_PATH}
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules
    ${LIBPLUGIN_DIR}/cmake)
find_package(CoppeliaSim 4.0.0.0 REQUIRED)
find_package(Python2 REQUIRED COMPONENTS Interpreter)

catkin_package(CATKIN_DEPENDS ${PKG_DEPS})

file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/generated)
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/external)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/generated)
include_directories(${catkin_INCLUDE_DIRS})
include_directories(${COPPELIASIM_INCLUDE_DIR})
include_directories(${COPPELIASIM_COMMON_DIR})

set(generatedFiles)
file(GLOB templateFiles RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/templates/ ${CMAKE_CURRENT_SOURCE_DIR}/templates/*)
foreach(templateFile ${templateFiles})
    add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/generated/${templateFile}" COMMAND ${Python2_EXECUTABLE} "${LIBPLUGIN_DIR}/simStubsGen/external/pycpp/pycpp.py" -p "messages_file=${CMAKE_CURRENT_SOURCE_DIR}/meta/messages.txt" -p "services_file=${CMAKE_CURRENT_SOURCE_DIR}/meta/services.txt" -i "${CMAKE_CURRENT_SOURCE_DIR}/templates/${templateFile}" -o "${CMAKE_CURRENT_BINARY_DIR}/generated/${templateFile}" -P "${CMAKE_CURRENT_SOURCE_DIR}/tools" DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/meta/messages.txt" "${CMAKE_CURRENT_SOURCE_DIR}/meta/services.txt" "${CMAKE_CURRENT_SOURCE_DIR}/templates/${templateFile}")
    set(generatedFiles ${generatedFiles} "${CMAKE_CURRENT_BINARY_DIR}/generated/${templateFile}")
endforeach()
add_custom_target(generate_ros_code DEPENDS ${generatedFiles})
coppeliasim_generate_stubs(${CMAKE_CURRENT_BINARY_DIR}/generated XML_FILE ${CMAKE_CURRENT_SOURCE_DIR}/meta/callbacks.xml LUA_FILE ${CMAKE_CURRENT_SOURCE_DIR}/simExtROSInterface.lua)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/include/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/config.h)

set(SOURCES
    src/sim_ros_interface.cpp
    src/ros_msg_builtin_io.cpp
    ${COPPELIASIM_EXPORTED_SOURCES}
    ${CMAKE_CURRENT_BINARY_DIR}/generated/ros_msg_io.cpp
    ${CMAKE_CURRENT_BINARY_DIR}/generated/ros_srv_io.cpp
)
add_library(simExtROSInterface ${SOURCES})
add_dependencies(simExtROSInterface generate_ros_code)
target_link_libraries(simExtROSInterface
    ${catkin_LIBRARIES}
    ${COPPELIASIM_LIBRARIES})

install(TARGETS simExtROSInterface DESTINATION ${COPPELIASIM_PLUGINS_DIR})

