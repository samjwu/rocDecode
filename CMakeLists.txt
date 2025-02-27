# ##############################################################################
# Copyright (c) 2023 - 2024 Advanced Micro Devices, Inc.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#
# ##############################################################################

cmake_minimum_required(VERSION 3.5)

set(VERSION "0.6.0")
set(CMAKE_CXX_STANDARD 17)

# Set Project Version and Language
project(rocdecode VERSION ${VERSION} LANGUAGES CXX)

set(CMAKE_INSTALL_LIBDIR "lib" CACHE STRING "Library install directory")
set(CMAKE_INSTALL_INCLUDEDIR "include" CACHE STRING "Include install directory")
include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})

find_program(MAKE_NSIS_EXE makensis)
find_program(RPMBUILD_EXE rpmbuild)
find_program(DPKG_EXE dpkg)

# ROCM Path
if(DEFINED ENV{ROCM_PATH})
  set(ROCM_PATH $ENV{ROCM_PATH} CACHE PATH "Default ROCm installation path")
elseif(ROCM_PATH)
  message("-- INFO:ROCM_PATH Set -- ${ROCM_PATH}")
else()
  set(ROCM_PATH /opt/rocm CACHE PATH "Default ROCm installation path")
endif()
# avoid setting the default installation path to /usr/local
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX ${ROCM_PATH} CACHE PATH "rocDecode default installation path" FORCE)
endif(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

set(DEFAULT_BUILD_TYPE "Release")
set(ENHANCED_MESSAGE ON)

if(ENHANCED_MESSAGE)
  string(ASCII 27 Esc)
  set(ColourReset "${Esc}[m")
  set(Red "${Esc}[31m")
  set(Green "${Esc}[32m")
  set(Yellow "${Esc}[33m")
  set(Blue "${Esc}[34m")
  set(BoldBlue "${Esc}[1;34m")
  set(Magenta "${Esc}[35m")
  set(Cyan "${Esc}[36m")
  set(White "${Esc}[37m")
endif()
message("-- ${BoldBlue}rocDecode Version -- ${VERSION}${ColourReset}")
message("-- ${BoldBlue}rocDecode Install Path -- ${CMAKE_INSTALL_PREFIX}${ColourReset}")

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH} ${ROCM_PATH}/hip)
set(CMAKE_CXX_COMPILER ${ROCM_PATH}/llvm/bin/clang++)

# rocDecode Default Options
option(BUILD_WITH_AMD_ADVANCE "Build rocDecode for advanced AMD GPU Architecture" OFF)

# rocDecode Build Type
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE STRING "rocDecode Default Build Type" FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif()
if(CMAKE_BUILD_TYPE MATCHES Debug)
  # -O0 -- Don't Optimize output file 
  # -g  -- generate debugging information, generate debugging information, dwarf-4 for making valgrind work
  # -Og -- Optimize for debugging experience rather than speed or size
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0 -gdwarf-4 -Og")
else()
  # -O3       -- Optimize output file 
  # -DNDEBUG  -- turn off asserts 
  # -fPIC     -- Generate position-independent code if possible
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -DNDEBUG -fPIC")
endif()
message("-- ${BoldBlue}rocDecode Build Type -- ${CMAKE_BUILD_TYPE}${ColourReset}")

find_package(HIP QUIET)
find_package(Libva QUIET)

if(HIP_FOUND AND Libva_FOUND)

  # HIP
  set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} hip::host)
  # LibVA
  include_directories(${LIBVA_INCLUDE_DIR})
  set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} ${LIBVA_LIBRARY})
  set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} ${LIBVA_DRM_LIBRARY})

  #filesystem: c++ compilers less than equal to 8.5 need explicit link with stdc++fs
  if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS_EQUAL "8.5")
    set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} stdc++fs)
  endif()

  # local include files
  include_directories(api src/rocdecode src/parser src/rocdecode/vaapi)
  # source files
  file(GLOB_RECURSE SOURCES "./src/*.cpp")
  # rocdecode.so
  add_library(${PROJECT_NAME} SHARED ${SOURCES})

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++17")
  target_link_libraries(${PROJECT_NAME} ${LINK_LIBRARY_LIST})

  set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)
  set_target_properties(${PROJECT_NAME} PROPERTIES LINKER_LANGUAGE CXX)
  set_target_properties(${PROJECT_NAME} PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})

  # install rocDecode libs -- {ROCM_PATH}/lib
  install(TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT runtime NAMELINK_SKIP)
  install(TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT dev NAMELINK_ONLY)
  install(TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT asan)
  # install rocDecode include files -- {ROCM_PATH}/include/rocdecode
  install(FILES api/rocdecode.h api/rocparser.h
          DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME} COMPONENT dev)
  # install rocDecode samples -- {ROCM_PATH}/share/rocdecode
  install(DIRECTORY cmake DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME} COMPONENT dev)
  install(DIRECTORY utils/rocvideodecode DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/utils COMPONENT dev)
  install(FILES samples/videoDecode/CMakeLists.txt samples/videoDecode/README.md samples/videoDecode/videodecode.cpp DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/videoDecode COMPONENT dev)
  install(FILES samples/videoDecodeMem/CMakeLists.txt samples/videoDecodeMem/README.md samples/videoDecodeMem/videodecodemem.cpp DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/videoDecodeMem COMPONENT dev)
  install(FILES samples/videoDecodePerf/CMakeLists.txt samples/videoDecodePerf/README.md samples/videoDecodePerf/videodecodeperf.cpp DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/videoDecodePerf COMPONENT dev)
  install(FILES samples/videoDecodeRGB/CMakeLists.txt samples/videoDecodeRGB/README.md samples/videoDecodeRGB/videodecrgb.cpp DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/videoDecodeRGB COMPONENT dev)
  install(FILES samples/videoDecodeBatch/CMakeLists.txt samples/videoDecodeBatch/README.md samples/videoDecodeBatch/videodecodebatch.cpp DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/videoDecodeBatch COMPONENT dev)
  install(FILES samples/common.h DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples COMPONENT dev)
  install(FILES utils/video_demuxer.h DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/utils COMPONENT dev)
  install(FILES utils/colorspace_kernels.cpp DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/utils COMPONENT dev)
  install(FILES utils/colorspace_kernels.h DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/utils COMPONENT dev)
  install(FILES utils/resize_kernels.cpp DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/utils COMPONENT dev)
  install(FILES utils/resize_kernels.h DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/utils COMPONENT dev)
  install(FILES utils/video_post_process.h DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/utils COMPONENT dev)
  install(FILES data/videos/AMD_driving_virtual_20-H265.mp4 data/videos/AMD_driving_virtual_20-H264.mp4 DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/video COMPONENT dev)
  # install license information - {ROCM_PATH}/share/doc/rocdecode
  set(CPACK_RESOURCE_FILE_LICENSE  "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR} COMPONENT runtime)
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR}-asan COMPONENT asan)
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR}-dev COMPONENT dev)
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR}-test COMPONENT test)
  # install test cmake
  install(FILES test/CMakeLists.txt DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/test COMPONENT test)
  install(DIRECTORY test/testScripts DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/test COMPONENT test)

  message("-- ${White}AMD ROCm rocDecode -- CMAKE_CXX_FLAGS:${CMAKE_CXX_FLAGS}${ColourReset}")
  message("-- ${White}AMD ROCm rocDecode -- Link Libraries: ${LINK_LIBRARY_LIST}${ColourReset}")

  # make test with CTest
  enable_testing()
  include(CTest)
  add_subdirectory(samples)

  # set package information
  set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
  set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
  set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
  set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
  set(CPACK_PACKAGE_LICENSE "MIT")
  # TBD: Change to generic support email
  set(CPACK_PACKAGE_CONTACT "rocDecode Support <mivisionx.support@amd.com>")
  set(CPACK_PACKAGE_VENDOR "AMD ROCm")
  set(CPACK_PACKAGE_GROUP "Development/Tools")
  set(CPACK_PACKAGE_HOMEPAGE "https://github.com/ROCm/rocDecode")

  if(ENABLE_ASAN_PACKAGING)
    set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "AMD rocDecode address sanitizer libraries")
    set(CPACK_RPM_PACKAGE_SUMMARY "AMD rocDecode address sanitizer libraries")
  else()
    set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "AMD rocDecode is a high performance video decode SDK for AMD GPUs")
    set(CPACK_RPM_PACKAGE_SUMMARY "AMD rocDecode is a high performance video decode SDK for AMD GPUs")
  endif()

  if(DEFINED ENV{ROCM_LIBPATCH_VERSION})
    set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION}.$ENV{ROCM_LIBPATCH_VERSION}")
  endif()

  # Find Ubuntu 22.04 - Add libstdc++-12-dev package deps for Dev Package
  file(READ "/etc/os-release" OS_RELEASE)
  string(REGEX MATCH "22.04" UBUNTU_22_FOUND ${OS_RELEASE})
  string(REGEX MATCH "SLES" SLES_FOUND ${OS_RELEASE})

  # Set the dependent packages - TBD: mesa-amdgpu-va-drivers should bring libdrm-amdgpu, and parts of mesa-amdgpu-dri-drivers but unavailable on RPM
  set(rocDecode_DEBIAN_PACKAGE_LIST  "rocm-hip-runtime, libva2, libdrm-amdgpu1, mesa-amdgpu-va-drivers")
  set(rocDecode_RPM_PACKAGE_LIST     "rocm-hip-runtime, libva, libdrm-amdgpu, mesa-amdgpu-va-drivers, mesa-amdgpu-dri-drivers")
  if(SLES_FOUND)
    set(rocDecode_RPM_PACKAGE_LIST     "rocm-hip-runtime, libva2, libdrm-amdgpu, mesa-amdgpu-va-drivers, mesa-amdgpu-dri-drivers")
  endif()
  # Set the dev dependent packages
  set(rocDecode_DEBIAN_DEV_PACKAGE_LIST  "rocm-hip-runtime-dev, libva-dev, pkg-config, ffmpeg, libavcodec-dev, libavformat-dev, libavutil-dev")
  if(UBUNTU_22_FOUND)
    set(rocDecode_DEBIAN_DEV_PACKAGE_LIST "${rocDecode_DEBIAN_DEV_PACKAGE_LIST}, libstdc++-12-dev")
  endif()
  # TBD - RPM packages need Fusion Packages - "ffmpeg, libavcodec-devel, libavformat-devel, libavutil-devel"
  set(rocDecode_RPM_DEV_PACKAGE_LIST  "rocm-hip-runtime-devel, libva-devel, pkg-config")
  
  # '%{?dist}' breaks manual builds on debian systems due to empty Provides
  execute_process(
    COMMAND rpm --eval %{?dist}
    RESULT_VARIABLE PROC_RESULT
    OUTPUT_VARIABLE EVAL_RESULT
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(PROC_RESULT EQUAL "0" AND NOT EVAL_RESULT STREQUAL "")
    string(APPEND CPACK_RPM_PACKAGE_RELEASE "%{?dist}")
  endif()

  # set package 
  set(CPACK_PACKAGING_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
  set(CPACK_DEBIAN_PACKAGE_HOMEPAGE ${CPACK_PACKAGE_HOMEPAGE})
  set(CPACK_RPM_PACKAGE_URL ${CPACK_PACKAGE_HOMEPAGE})
  set(CPACK_RPM_PACKAGE_AUTOREQPROV "no")
  # Debian package
  set(CPACK_DEB_COMPONENT_INSTALL ON)
  set(CPACK_DEBIAN_RUNTIME_PACKAGE_NAME "${PROJECT_NAME}")
  set(CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS "rocm-core, ${rocDecode_DEBIAN_PACKAGE_LIST}")
  set(CPACK_DEBIAN_DEV_PACKAGE_NAME "${PROJECT_NAME}-dev")
  set(CPACK_DEBIAN_DEV_PACKAGE_DEPENDS 
  "rocm-core, ${PROJECT_NAME}, ${rocDecode_DEBIAN_DEV_PACKAGE_LIST}")
  # Debian package - specific variable for ASAN
  set(CPACK_DEBIAN_ASAN_PACKAGE_NAME "${PROJECT_NAME}-asan" )
  set(CPACK_DEBIAN_ASAN_PACKAGE_DEPENDS "rocm-core-asan, ${rocDecode_DEBIAN_PACKAGE_LIST}" )
  # Debian package - Test
  set(CPACK_DEBIAN_TEST_PACKAGE_NAME "${PROJECT_NAME}-test" )
  set(CPACK_DEBIAN_TEST_PACKAGE_DEPENDS "rocm-core, ${CPACK_DEBIAN_DEV_PACKAGE_NAME}" )
  # RPM package
  set(CPACK_RPM_COMPONENT_INSTALL ON)
  set(CPACK_RPM_RUNTIME_PACKAGE_NAME "${PROJECT_NAME}")
  set(CPACK_RPM_RUNTIME_PACKAGE_REQUIRES "rocm-core, ${rocDecode_RPM_PACKAGE_LIST}")
  set(CPACK_RPM_RUNTIME_PACKAGE_PROVIDES "${PROJECT_NAME}")
  set(CPACK_RPM_RUNTIME_PACKAGE_OBSOLETES "${PROJECT_NAME}")
  set(CPACK_RPM_DEV_PACKAGE_NAME "${PROJECT_NAME}-devel")
  set(CPACK_RPM_DEV_PACKAGE_REQUIRES "rocm-core, ${PROJECT_NAME}, ${rocDecode_RPM_DEV_PACKAGE_LIST}")
  set(CPACK_RPM_DEV_PACKAGE_PROVIDES "${PROJECT_NAME}-devel")
  set(CPACK_RPM_DEV_PACKAGE_OBSOLETES "${PROJECT_NAME}-devel")
  set(CPACK_RPM_PACKAGE_LICENSE "MIT" )
  # RPM package specific variable for ASAN
  set(CPACK_RPM_ASAN_PACKAGE_NAME "${PROJECT_NAME}-asan" )
  set(CPACK_RPM_ASAN_PACKAGE_REQUIRES "rocm-core-asan, ${rocDecode_RPM_PACKAGE_LIST}" )
  set(CPACK_RPM_ASAN_PACKAGE_PROVIDES "${PROJECT_NAME}-asan")
  set(CPACK_RPM_ASAN_PACKAGE_OBSOLETES "${PROJECT_NAME}-asan")
  # RPM package specific variable for Test
  set(CPACK_RPM_TEST_PACKAGE_NAME "${PROJECT_NAME}-test" )
  set(CPACK_RPM_TEST_PACKAGE_REQUIRES "rocm-core, ${CPACK_RPM_DEV_PACKAGE_NAME}" )
  set(CPACK_RPM_TEST_PACKAGE_PROVIDES "${PROJECT_NAME}-test")
  set(CPACK_RPM_TEST_PACKAGE_OBSOLETES "${PROJECT_NAME}-test")

  if(NOT ROCM_DEP_ROCMCORE)
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_RPM_RUNTIME_PACKAGE_REQUIRES ${CPACK_RPM_RUNTIME_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core-asan," "" CPACK_RPM_ASAN_PACKAGE_REQUIRES ${CPACK_RPM_ASAN_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_RPM_DEV_PACKAGE_REQUIRES ${CPACK_RPM_DEV_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_RPM_TEST_PACKAGE_REQUIRES ${CPACK_RPM_TEST_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS ${CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS})
    string(REGEX REPLACE ",? ?rocm-core-asan," "" CPACK_DEBIAN_ASAN_PACKAGE_DEPENDS ${CPACK_DEBIAN_ASAN_PACKAGE_DEPENDS})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_DEBIAN_DEV_PACKAGE_DEPENDS ${CPACK_DEBIAN_DEV_PACKAGE_DEPENDS})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_DEBIAN_TEST_PACKAGE_DEPENDS ${CPACK_DEBIAN_TEST_PACKAGE_DEPENDS})
  endif()

  if(ENABLE_ASAN_PACKAGING)
    # ASAN Package requires asan component with only libraries and license file
    set(CPACK_COMPONENTS_ALL asan)
  else()
    set(CPACK_COMPONENTS_ALL runtime dev test)
  endif()

  set(CPACK_DEBIAN_FILE_NAME "DEB-DEFAULT")
  set(CPACK_RPM_FILE_NAME "RPM-DEFAULT")
  set(CPACK_DEBIAN_PACKAGE_RELEASE "local")
  set(CPACK_RPM_PACKAGE_RELEASE "local")

  if(DEFINED ENV{CPACK_DEBIAN_PACKAGE_RELEASE})
    set(CPACK_DEBIAN_PACKAGE_RELEASE $ENV{CPACK_DEBIAN_PACKAGE_RELEASE})
  endif()
  if(DEFINED ENV{CPACK_RPM_PACKAGE_RELEASE})
    set(CPACK_RPM_PACKAGE_RELEASE $ENV{CPACK_RPM_PACKAGE_RELEASE})
  endif()

  set(CPACK_GENERATOR "TGZ;ZIP")
  if(EXISTS ${MAKE_NSIS_EXE})
    list(APPEND CPACK_GENERATOR "NSIS")
  endif()
  if(EXISTS ${RPMBUILD_EXE})
    list(APPEND CPACK_GENERATOR "RPM")
    message("-- ${White}rocDecode .rpm RunTime Package Requirements -- ${CPACK_RPM_RUNTIME_PACKAGE_REQUIRES}${ColourReset}")
    message("-- ${White}rocDecode .rpm Devel Package Requirements -- ${CPACK_RPM_DEV_PACKAGE_REQUIRES}${ColourReset}")
  endif()
  if(EXISTS ${DPKG_EXE})
    list(APPEND CPACK_GENERATOR "DEB")
    message("-- ${White}rocDecode .deb RunTime Package Requirements -- ${CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS}${ColourReset}")
    message("-- ${White}rocDecode .deb Dev Package Requirements -- ${CPACK_DEBIAN_DEV_PACKAGE_DEPENDS}${ColourReset}")
  endif()

  include(CPack)

  cpack_add_component(runtime
                    DISPLAY_NAME "rocDecode Runtime Package"
                    DESCRIPTION "AMD rocDecode is a high performance video decode SDK for AMD GPUs. \
rocDecode runtime package provides rocDecode library and license.txt")

  cpack_add_component(dev
                    DISPLAY_NAME "rocDecode Develop Package"
                    DESCRIPTION "AMD rocDecode is a high performance video decode SDK for AMD GPUs. \
rocDecode develop package provides rocDecode library, header files, samples, and license.txt")

  cpack_add_component(asan
                    DISPLAY_NAME "rocDecode ASAN Package"
                    DESCRIPTION "AMD rocDecode is a high performance video decode SDK for AMD GPUs. \
rocDecode ASAN package provides rocDecode ASAN libraries")

  cpack_add_component(test
                  DISPLAY_NAME "rocDecode Test Package"
                  DESCRIPTION "AMD rocDecode is a high performance video decode SDK for AMD GPUs. \
rocDecode Test package provides rocDecode Test Components")

else()
  message("-- ${Red}AMD ROCm rocDecode -- unmet dependencies${ColourReset}")
  if(NOT HIP_FOUND)
    message(FATAL_ERROR "-- ERROR!: HIP Not Found! - please install rocm-hip-runtime-dev!")
  endif()
  if(NOT Libva_FOUND)
    message(FATAL_ERROR "-- ERROR!: libva Not Found - please install libva-dev/libva-devel!")
  endif()
endif()
