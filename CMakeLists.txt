cmake_minimum_required(VERSION 2.8)

project(kacanopen C CXX)

find_package(Boost 1.46.1 COMPONENTS system filesystem REQUIRED)
find_package(Threads)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra") # -Werror
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra") # -Werror

set(SDO_RESPONSE_TIMEOUT_MS "2000")
set(CONSECUTIVE_SEND_PAUSE_MS "2")

add_definitions("-DSDO_RESPONSE_TIMEOUT_MS=${SDO_RESPONSE_TIMEOUT_MS}")
message(STATUS "SDO response timeout is set to ${SDO_RESPONSE_TIMEOUT_MS}ms")

add_definitions("-DCONSECUTIVE_SEND_PAUSE_MS=${CONSECUTIVE_SEND_PAUSE_MS}")
message(STATUS "Pause between two consecutively sent CAN frames is set to ${CONSECUTIVE_SEND_PAUSE_MS}ms")

# TODO(mvukov) Install EDS files to appropriate place.
add_definitions("-DSHARE_SOURCE_PATH=\"${CMAKE_CURRENT_LIST_DIR}/master/share\"")
add_definitions("-DSHARE_INSTALLED_PATH=\"${CMAKE_INSTALL_PREFIX}/share/kacanopen\"")
message(STATUS "SHARE_SOURCE_PATH set to ${CMAKE_CURRENT_LIST_DIR}/share.")
message(STATUS "SHARE_INSTALLED_PATH set to ${CMAKE_INSTALL_PREFIX}/share/kacanopen.")

include_directories(
  core/include
  drivers_lgpl/include
  drivers_lgpl/include/timers_unix
  drivers_lgpl/include/unix
  master/include
  ${Boost_INCLUDE_DIRS})

file(GLOB DRIVER_SRC "drivers_lgpl/socket/*.c")
add_library(kacanopen_can_socket SHARED ${DRIVER_SRC})

file(GLOB CORE_SRC "core/src/*.cpp")
add_library(kacanopen_core SHARED ${CORE_SRC})
target_link_libraries(kacanopen_core
  kacanopen_can_socket ${CMAKE_DL_LIBS} ${CMAKE_THREAD_LIBS_INIT})

file(GLOB MASTER_SRC "master/src/*.cpp")
add_library(kacanopen_master SHARED ${MASTER_SRC})
target_link_libraries(kacanopen_master
  kacanopen_core ${Boost_SYSTEM_LIBRARY} ${Boost_FILESYSTEM_LIBRARY}
  ${CMAKE_THREAD_LIBS_INIT})

set(kacanopen_LIBRARIES kacanopen_can_socket kacanopen_core kacanopen_master)

install(TARGETS ${kacanopen_LIBRARIES}
  EXPORT kacanopenTargets
  LIBRARY DESTINATION lib
)

install(DIRECTORY core/include/
   DESTINATION include/kacanopen
   FILES_MATCHING PATTERN "*.h"
)

install(DIRECTORY master/include/
   DESTINATION include/kacanopen
   FILES_MATCHING PATTERN "*.h"
)

# Add all targets to the build-tree export set
export(TARGETS ${kacanopen_LIBRARIES}
  FILE "${PROJECT_BINARY_DIR}/kacanopenTargets.cmake"
)
 
# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
export(PACKAGE kacanopen)

configure_file(kacanopen-config.cmake.in
  ${PROJECT_BINARY_DIR}/kacanopen-config.cmake @ONLY
)
configure_file(kacanopen-config-version.cmake.in
  ${PROJECT_BINARY_DIR}/kacanopen-config-version.cmake @ONLY
)
 
install(FILES
  ${PROJECT_BINARY_DIR}/kacanopen-config.cmake
  ${PROJECT_BINARY_DIR}/kacanopen-config-version.cmake
  DESTINATION share/kacanopen)
 
# Install the export set for use with the install-tree.
install(EXPORT kacanopenTargets DESTINATION share/kacanopen)

# Install EDS files.
install(DIRECTORY master/share/eds_library
   DESTINATION share/kacanopen
)

install(FILES package.xml DESTINATION share/kacanopen)
