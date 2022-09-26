############################################################################
# Copyright (c) Johan Mabille, Sylvain Corlay, Wolf Vollprecht and         #
# Martin Renou                                                             #
# Copyright (c) QuantStack                                                 #
# Copyright (c) Serge Guelton                                              #
#                                                                          #
# Distributed under the terms of the BSD 3-Clause License.                 #
#                                                                          #
# The full license is in the file LICENSE, distributed with this software. #
############################################################################

cmake_minimum_required(VERSION 3.1)
project(xsimd-algorithm)
set(XSIMD_ALGORITHM_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)

# Versioning
# ==========

file(STRINGS "${XSIMD_ALGORITHM_INCLUDE_DIR}/xsimd-algorithm/xversion.hpp" xsimd_version_defines
    REGEX "#define XSIMD_ALGORITHM_VERSION_(MAJOR|MINOR|PATCH)")
foreach(ver ${xsimd_version_defines})
    if(ver MATCHES "#define XSIMD_ALGORITHM_VERSION_(MAJOR|MINOR|PATCH) +([^ ]+)$")
        set(XSIMD_ALGORITHM_VERSION_${CMAKE_MATCH_1} "${CMAKE_MATCH_2}" CACHE INTERNAL "")
    endif()
endforeach()
set(${PROJECT_NAME}_VERSION
    ${XSIMD_ALGORITHM_VERSION_MAJOR}.${XSIMD_ALGORITHM_VERSION_MINOR}.${XSIMD_ALGORITHM_VERSION_PATCH})
message(STATUS "xsimd-algorithm v${${PROJECT_NAME}_VERSION}")

# Dependencies
# ============

set(xsimd_REQUIRED_VERSION 10.0.0)
if(TARGET xsimd)
    set(xsimd_VERSION ${XSIMD_VERSION_MAJOR}.${XSIMD_VERSION_MINOR}.${XSIMD_VERSION_PATCH})
    # Note: This is not SEMVER compatible comparison
    if(${xsimd_VERSION} VERSION_LESS ${xsimd_REQUIRED_VERSION})
        message(ERROR "Mismatch xsimd versions. Found '${xsimd_VERSION}' but requires: '${xsimd_REQUIRED_VERSION}'")
    else()
        message(STATUS "Found xsimd v${xsimd_VERSION}")
    endif()
else()
    find_package(xsimd ${xsimd_REQUIRED_VERSION} REQUIRED)
    message(STATUS "Found xsimd: ${xsimd_INCLUDE_DIRS}/xsimd")
endif()

# Build
# =====

set(XSIMD_ALGORITHM_HEADERS
    ${XSIMD_ALGORITHM_INCLUDE_DIR}/xsimd-algorithm/xreduce.hpp
    ${XSIMD_ALGORITHM_INCLUDE_DIR}/xsimd-algorithm/xsimd-algorithm.hpp
    ${XSIMD_ALGORITHM_INCLUDE_DIR}/xsimd-algorithm/xtransform.hpp
)

add_library(xsimd-algorithm INTERFACE)

target_include_directories(xsimd-algorithm INTERFACE
    $<BUILD_INTERFACE:${XSIMD_ALGORITHM_INCLUDE_DIR}>
    $<INSTALL_INTERFACE:include>)

target_link_libraries(xsimd-algorithm INTERFACE xsimd)

option(BUILD_TESTS "xsimd-algorithm test suite" OFF)

if (BUILD_TESTS)
    enable_testing()
    add_subdirectory(test)
endif ()

# Installation
# ============

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(JoinPaths)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

install(TARGETS xsimd-algorithm
        EXPORT ${PROJECT_NAME}-targets)

# Makes the project importable from the build directory
export(EXPORT ${PROJECT_NAME}-targets
       FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake")

install(FILES ${XSIMD_ALGORITHM_HEADERS}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/xsimd-algorithm)
     
# GNUInstallDirs "DATADIR" wrong here; CMake search path wants "share".
set(XSIMD_CMAKECONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}" CACHE STRING "install path for xsimdConfig.cmake")

configure_package_config_file(${PROJECT_NAME}Config.cmake.in
                              "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
                              INSTALL_DESTINATION ${XSIMD_CMAKECONFIG_INSTALL_DIR})

# xsimd is header-only and does not depend on the architecture.
# Remove CMAKE_SIZEOF_VOID_P from xtensorConfigVersion.cmake so that an xtensorConfig.cmake
# generated for a 64 bit target can be used for 32 bit targets and vice versa.
set(_XTENSOR_CMAKE_SIZEOF_VOID_P ${CMAKE_SIZEOF_VOID_P})
unset(CMAKE_SIZEOF_VOID_P)
write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
                                 VERSION ${${PROJECT_NAME}_VERSION}
                                 COMPATIBILITY SameMajorVersion)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
        DESTINATION ${XSIMD_CMAKECONFIG_INSTALL_DIR})
install(EXPORT ${PROJECT_NAME}-targets
        FILE ${PROJECT_NAME}Targets.cmake
        DESTINATION ${XSIMD_CMAKECONFIG_INSTALL_DIR})

join_paths(libdir_for_pc_file "\${exec_prefix}" "${CMAKE_INSTALL_LIBDIR}")
join_paths(includedir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_INCLUDEDIR}")
configure_file(${PROJECT_NAME}.pc.in
               "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc"
                @ONLY)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig/")
