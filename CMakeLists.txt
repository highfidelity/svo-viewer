cmake_minimum_required(VERSION 2.8.12)

set(HIFI_DIR "${CMAKE_CURRENT_SOURCE_DIR}/hifi")
set(MACRO_DIR "${HIFI_DIR}/cmake/macros")

set(CMAKE_AUTOMOC ON)

set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} $ENV{QT_CMAKE_PREFIX_PATH})

# setup for find modules
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/hifi/cmake/modules/")

set(TARGET_NAME svo-viewer)
project(${TARGET_NAME})

if (APPLE)
  set(GL_HEADERS "#include <GLUT/glut.h>\n#include <OpenGL/glext.h>")
endif ()

if (UNIX AND NOT APPLE)
  # include the right GL headers for UNIX
  set(GL_HEADERS "#include <GL/gl.h>\n#include <GL/glut.h>\n#include <GL/glext.h>")
endif ()

if (WIN32)
  add_definitions(-D_USE_MATH_DEFINES ) # apparently needed to get M_PI and other defines from cmath/math.h
  add_definitions(-DWINDOWS_LEAN_AND_MEAN ) # needed to make sure windows doesn't go to crazy with its defines
  
  set(GL_HEADERS "#include <windowshacks.h>\n#include <GL/glew.h>\n#include <GL/glut.h>")
endif ()

# create the ${TARGET_NAME}-config.h file based on GL_HEADERS above
configure_file(${TARGET_NAME}-config.h.in "${PROJECT_BINARY_DIR}/includes/${TARGET_NAME}-config.h")
configure_file(${TARGET_NAME}-version.h.in "${PROJECT_BINARY_DIR}/includes/${TARGET_NAME}-version.h")

# grab the implementation and header files from src dirs
file(GLOB APPLICATION_SRCS src/*.c src/*.cpp src/*.h)

find_package(Qt5 COMPONENTS Core Gui Multimedia Network OpenGL Script Svg WebKit WebKitWidgets Xml UiTools)

if (APPLE)
  # configure CMake to use a custom Info.plist
  SET_TARGET_PROPERTIES(${this_target} PROPERTIES MACOSX_BUNDLE_INFO_PLIST MacOSXBundleInfo.plist.in )

  set(MACOSX_BUNDLE_BUNDLE_NAME svo-viewer)
  set(MACOSX_BUNDLE_GUI_IDENTIFIER io.highfidelity.${TARGET_NAME})
endif ()

# create the executable, make it a bundle on OS X
add_executable(${TARGET_NAME} MACOSX_BUNDLE ${APPLICATION_SRCS})

# set up the external glm library
find_package(GLM REQUIRED)
include_directories("${GLM_INCLUDE_DIRS}")

if (APPLE OR UNIX)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -isystem ${GLM_INCLUDE_DIRS}")
endif ()

# link in the hifi shared library
include(${MACRO_DIR}/LinkHifiLibrary.cmake)

# link required hifi libraries
link_hifi_library(shared ${TARGET_NAME} "hifi")
link_hifi_library(octree ${TARGET_NAME} "hifi")
link_hifi_library(voxels ${TARGET_NAME} "hifi")

# include headers for svo-viewer
include_directories("${PROJECT_SOURCE_DIR}/src" "${PROJECT_BINARY_DIR}/includes")

target_link_libraries(${TARGET_NAME} "${ZLIB_LIBRARIES}" 
  Qt5::Core Qt5::Gui Qt5::Network Qt5::OpenGL 
  Qt5::Script Qt5::Svg Qt5::WebKit Qt5::WebKitWidgets Qt5::Xml Qt5::UiTools)

if (APPLE)
  # link in required OS X frameworks and include the right GL headers
  find_library(AppKit AppKit)
  find_library(CoreAudio CoreAudio)
  find_library(CoreServices CoreServices)
  find_library(Carbon Carbon)
  find_library(Foundation Foundation)
  find_library(GLUT GLUT)
  find_library(OpenGL OpenGL)
  find_library(IOKit IOKit)
  find_library(QTKit QTKit)
  find_library(QuartzCore QuartzCore)

  target_link_libraries(
    ${TARGET_NAME}
    ${AppKit}
    ${CoreAudio}
    ${CoreServices}
    ${Carbon}
    ${Foundation}
    ${GLUT}
    ${OpenGL}
    ${IOKit}
    ${QTKit}
    ${QuartzCore}
  )
  
  # install command for OS X bundle
  INSTALL(TARGETS ${TARGET_NAME}
    BUNDLE DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/install" COMPONENT Runtime
    RUNTIME DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/install" COMPONENT Runtime
  )
else ()
  find_package(OpenGL REQUIRED)
  find_package(GLUT REQUIRED)
  
  include_directories(SYSTEM "${GLUT_INCLUDE_DIR}")
  
  if (${OPENGL_INCLUDE_DIR})
    include_directories(SYSTEM "${OPENGL_INCLUDE_DIR}")
  endif ()
  
  target_link_libraries(${TARGET_NAME} "${OPENGL_LIBRARY}" "${GLUT_LIBRARIES}")
  
  # link target to external libraries
  if (WIN32)
    find_package(GLEW REQUIRED)
    include_directories(${GLEW_INCLUDE_DIRS})
    
    # we're using static GLEW, so define GLEW_STATIC
    add_definitions(-DGLEW_STATIC)
    
    target_link_libraries(${TARGET_NAME} "${GLEW_LIBRARY}" wsock32.lib opengl32.lib)
  endif()
endif ()
