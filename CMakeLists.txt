project("OpenPHT")
cmake_minimum_required(VERSION 3.1.0 FATAL_ERROR)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Debug' as none was specified.")
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo")
endif()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(root ${PROJECT_SOURCE_DIR})
set(plexdir ${root}/plex)
set(CMAKE_MODULE_PATH ${plexdir}/CMakeModules ${CMAKE_MODULE_PATH})

include(GetGitRevisionDescription)
include(GetDate)
today(PLEX_CURRENT_DATE)
get_git_head_revision(REFSPEC PLEX_FULL_GIT_REVISION)
if(PLEX_FULL_GIT_REVISION STREQUAL "GITDIR-NOTFOUND")
  if(EXISTS ${root}/GitRevision.txt)
    file(STRINGS "${root}/GitRevision.txt" PLEX_GIT_REVISION)
  else()
    set(PLEX_GIT_REVISION "git")
  endif()
else()
  string(SUBSTRING ${PLEX_FULL_GIT_REVISION} 0 8 PLEX_GIT_REVISION)
endif()

set(PLEX_VERSION_MAJOR 1)
set(PLEX_VERSION_MINOR 6)
set(PLEX_VERSION_PATCH 2)
if(DEFINED ENV{BUILD_NUMBER})
  set(PLEX_VERSION_BUILD "$ENV{BUILD_NUMBER}")
  set(PLEX_VERSION_BUILD_NR "$ENV{BUILD_NUMBER}")
else()
  set(PLEX_VERSION_BUILD "dev")
  set(PLEX_VERSION_BUILD_NR "0")
endif()
set(PLEX_VERSION_STRING ${PLEX_VERSION_MAJOR}.${PLEX_VERSION_MINOR}.${PLEX_VERSION_PATCH}.${PLEX_VERSION_BUILD}-${PLEX_GIT_REVISION})
set(PLEX_VERSION_STRING_SHORT ${PLEX_VERSION_MAJOR}.${PLEX_VERSION_MINOR}.${PLEX_VERSION_PATCH})
set(PLEX_VERSION_STRING_SHORT_BUILD ${PLEX_VERSION_STRING_SHORT}.${PLEX_VERSION_BUILD})
set(PLEX_WINDOWS_VERSION ${PLEX_VERSION_MAJOR},${PLEX_VERSION_MINOR},${PLEX_VERSION_PATCH},${PLEX_VERSION_BUILD_NR})
set(XBMC_VERSION "Frodo")

configure_file(${plexdir}/git_revision.h.in git_revision.h)

message(STATUS "Building OpenPHT, version: ${PLEX_VERSION_STRING}")

include(CMakeFunctions)

# Now check for available functions
include(CMakeConfig)

# Now build auxilary xbmc libraries.
# Note that we want to do this before we add all
# our defines and includes, otherwise they will
# carry over
add_subdirectory(lib)

include_directories(
  ${CMAKE_CURRENT_BINARY_DIR}
  ${CMAKE_BINARY_DIR}/xbmc
  ${root}/plex/Third-Party/gtest/include
  ${root}/plex/Third-Party/gmock/include

  ${dependdir}/include
  ${dependdir}/include/libcec
  ${dependdir}/include/freetype2
  ${dependdir}/include/python2.6
  ${dependdir}/include/python2.7
  ${ffmpegdir}/include
  ${root}/xbmc
)

# Platfrom includes
if(TARGET_POSIX)
  include_directories(${root}/xbmc/linux)
endif(TARGET_POSIX)

if(TARGET_COMMON_DARWIN)
  include_directories(${root}/xbmc/osx)
endif(TARGET_COMMON_DARWIN)

if(TARGET_WIN32)
  include_directories(
    ${root}/xbmc/win32
    ${root}/lib/ffmpeg
    ${root}/lib/ffmpeg/include-xbmc-win32
    ${root}/lib/win32
    "C:/Program Files (x86)/Microsoft DirectX SDK (June 2010)/Include"
  )
endif(TARGET_WIN32)

if(NOT OPENELEC)
  # Build the texturepacker
  add_subdirectory(tools/TexturePacker)
endif()

# add our visualizations
add_subdirectory(xbmc/visualizations)

# global includes
include_directories(
  ${plexdir}
  ${plexdir}/Utility
  ${plexdir}/Owned

  ${root}/xbmc/utils
  ${root}
  ${root}/xbmc/cores/dvdplayer
  ${root}/xbmc/settings
  ${root}/lib
  ${CMAKE_BINARY_DIR}/lib/cpluff/cpluff-prefix/src/cpluff/libcpluff
  
  ${root}/xbmc/cores/AudioEngine
)

if(ENABLE_PYTHON)
    if(EXTERNAL_PYTHON_HOME)
      if(TARGET_LINUX OR TARGET_OSX OR TARGET_RPI OR TARGET_FREEBSD)
        install(DIRECTORY ${EXTERNAL_PYTHON_HOME}/lib/python${PYTHON_VERSION} DESTINATION ${LIBPATH}/lib/ COMPONENT RUNTIME
          PATTERN python${PYTHON_VERSION}/test EXCLUDE
          REGEX .*\\.pyc$ EXCLUDE
          REGEX .*\\.pyo$ EXCLUDE
        )
        install(FILES ${PYTHON_INCLUDE_DIRS}/pyconfig.h DESTINATION "${LIBPATH}/include/python${PYTHON_VERSION}" COMPONENT RUNTIME)
      endif()
    elseif()
      if(TARGET_LINUX OR TARGET_OSX OR TARGET_FREEBSD)
        install(DIRECTORY /usr/lib/python${PYTHON_VERSION} DESTINATION ${LIBPATH}/lib/ COMPONENT RUNTIME
          PATTERN python${PYTHON_VERSION}/test EXCLUDE
          REGEX .*\\.pyc$ EXCLUDE
          REGEX .*\\.pyo$ EXCLUDE
        )
        install(FILES ${PYTHON_INCLUDE_DIRS}/pyconfig.h DESTINATION "${LIBPATH}/include/python${PYTHON_VERSION}" COMPONENT RUNTIME)
      endif()
    endif()
endif()

# UPnP includes
include_directories(
  ${root}/lib/libUPnP/Platinum/Source/Platinum
  ${root}/lib/libUPnP/Platinum/Source/Core
  ${root}/lib/libUPnP/Platinum/Source/Extras
  ${root}/lib/libUPnP/Platinum/Source/Devices/MediaConnect
  ${root}/lib/libUPnP/Platinum/Source/Devices/MediaRenderer
  ${root}/lib/libUPnP/Platinum/Source/Devices/MediaServer
  ${root}/lib/libUPnP/Neptune/Source/Core
  ${root}/lib/libUPnP/Neptune/Source/System/Posix
)

add_subdirectory(xbmc)
if(TARGET_OSX)
  add_subdirectory(tools/EventClients/Clients/OSXRemote)
endif()
add_subdirectory(plex)

if(XRANDR_FOUND)
  add_executable(xbmc-xrandr xbmc-xrandr.c)
  target_link_libraries(xbmc-xrandr ${CONFIG_LIBRARY_XRANDR} ${CONFIG_LIBRARY_X11} -lm)
  install(TARGETS xbmc-xrandr DESTINATION ${BINPATH})
endif()
  
include(CPackConfig)
