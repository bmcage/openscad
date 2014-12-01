# instructions - see ../doc/testing.txt

set(DEBUG_OSCD 1) # print debug info during cmake

SET(CMAKE_BUILD_TYPE Debug)
#SET(CMAKE_BUILD_TYPE Release)

project(openscad_nogui)


# User settable Target options (remove build directory on changes!)
option (NOHARFBUZZ      "Compile without HarfBuzz"      ON)


cmake_minimum_required(VERSION 2.8)
if("${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}" VERSION_GREATER 2.8.3)
  # Explicitly use new include policy to avoid globally shadowing included modules
  # http://www.cmake.org/cmake/help/cmake-2-8-docs.html#policy:CMP0017
  cmake_policy(SET CMP0017 NEW)
endif()

# Needed for cmake < 2.8.3
include(CMakeParseArguments.cmake)

# Detect Lion and force gcc
IF (APPLE)
   EXECUTE_PROCESS(COMMAND sw_vers -productVersion OUTPUT_VARIABLE MACOSX_VERSION)
   IF (NOT ${MACOSX_VERSION} VERSION_LESS "10.9.0")
     message("Detected Maverick (10.9) or later")
     set(CMAKE_C_COMPILER "clang")
     set(CMAKE_CXX_COMPILER "clang++")
     # Somehow, since we build dependencies for 10.7, we need to also build executables
     # for 10.7. This used to not be necessary, but since 10.9 it apparently is..
     SET(CMAKE_OSX_DEPLOYMENT_TARGET 10.7 CACHE STRING "Deployment target")
   ELSEIF (NOT ${MACOSX_VERSION} VERSION_LESS "10.8.0")
     message("Detected Mountain Lion (10.8)")
     set(CMAKE_C_COMPILER "clang")
     set(CMAKE_CXX_COMPILER "clang++")
   ELSEIF (NOT ${MACOSX_VERSION} VERSION_LESS "10.7.0")
     message("Detected Lion (10.7)")
     set(CMAKE_C_COMPILER "clang")
     set(CMAKE_CXX_COMPILER "clang++")
   ELSE()
     message("Detected Snow Leopard (10.6) or older")
     if (USE_LLVM)
       message("Using LLVM compiler")
       set(CMAKE_C_COMPILER "llvm-gcc")
       set(CMAKE_CXX_COMPILER "llvm-g++")
     endif()
   ENDIF()
ENDIF(APPLE)


set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}" ${CMAKE_SOURCE_DIR}/../tests)

# Build debug build as default
if(NOT CMAKE_BUILD_TYPE)
  #  set(CMAKE_BUILD_TYPE Release)
  if(CMAKE_COMPILER_IS_GNUCXX)
    execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpversion OUTPUT_VARIABLE GCC_VERSION)
    if (GCC_VERSION VERSION_GREATER 4.6)
      set(CMAKE_BUILD_TYPE RelWithDebInfo)
    else()
      set(CMAKE_BUILD_TYPE Debug)
    endif()
  else()
    set(CMAKE_BUILD_TYPE RelWithDebInfo)
  endif()
endif()
message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")

if(CMAKE_COMPILER_IS_GNUCXX)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-strict-aliasing")
endif()

if(${CMAKE_BUILD_TYPE} STREQUAL "Debug")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DDEBUG")
endif()

# MCAD
if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/../libraries/MCAD/__init__.py)
  #message(FATAL_ERROR "MCAD not found. You can install from the OpenSCAD root as follows: \n  git submodule update --init")
  message(STATUS "MCAD not found, so you cannot use it. You can install from the OpenSCAD root as follows: \n  git submodule update --init")
endif()

# NULLGL is set by default
#- Allow us to build without OpenGL(TM). run 'cmake .. -DNULLGL=1'
set(NULLGL 1)
if(NULLGL)
  set(ENABLE_OPENCSG_FLAG "") # OpenCSG is entirely an OpenGL software
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNULLGL")
  set(SKIP_IMAGEMAGICK "1") # we dont generate png, so nothing to compare
else()
  set(ENABLE_OPENCSG_FLAG "-DENABLE_OPENCSG")
endif()

#
# Windows
#

if(WIN32 AND MSVC)
  set(WIN32_STATIC_BUILD "True")
endif()

if(WIN32_STATIC_BUILD AND MSVC)
  if(${CMAKE_BUILD_TYPE} STREQUAL "Debug")
    set(EMSG "\nTo build Win32 STATIC OpenSCAD please see doc/testing.txt")
    message(FATAL_ERROR ${EMSG})
  endif()
endif()

# Disable warnings
if(WIN32 AND MSVC)
  # too long decorated names
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4503")
  # int cast to bool in CGAL
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4800")
  # unreferenced parameters in CGAL
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4100")
  # fopen_s advertisement
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_SECURE_NO_DEPRECATE")
  # lexer uses strdup & other POSIX stuff
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_NONSTDC_NO_DEPRECATE")
  # M_PI
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_USE_MATH_DEFINES")
endif()

# Debugging - if you uncomment, use nmake -f Makefile > log.txt (the log is big)
if(WIN32 AND MSVC)
  # Linker debugging
  #set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -VERBOSE")

  # Compiler debugging
  # you have to pass -DCMAKE_VERBOSE_MAKEFILE=ON to cmake when you run it. 
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
  if (WIN32 OR ${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fpermissive -frounding-math")
  endif()
endif()

# Clang compiler

if(${CMAKE_CXX_COMPILER} MATCHES ".*clang.*")
  # disable enormous amount of warnings about CGAL
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-parameter")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-variable")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-function")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-sign-compare")
endif()

#
# Build test apps
#

function(inclusion user_set_path found_paths)
  # Set up compiler include paths with prepend/append rules. Input is 
  # a path and a set of paths. If user_set_path matches anything in found_paths
  # then we prepend the found_paths because we assume the user wants
  # their set_paths to be a priority. 

  if (DEBUG_OSCD)
    message(STATUS "inclusion:")
    message(STATUS "  ${user_set_path}: ${${user_set_path}}")
    message(STATUS "  ${found_paths}: ${${found_paths}}")
  endif()
  set(inclusion_match 0)
  if (${user_set_path})
    foreach(found_path ${${found_paths}})
      string(FIND ${found_path} ${${user_set_path}} INDEX)
      if (DEFINED INDEX)
        if (${INDEX} EQUAL 0)
          set(inclusion_match 1)
        endif()
      endif()
    endforeach()
    if (inclusion_match)
      include_directories(BEFORE ${${found_paths}})
      if (DEBUG_OSCD)
        message(STATUS "inclusion prepend ${${found_paths}} for ${user_set_path}")
      endif()
      set(inclusion_match 0)
    endif()
  endif()
  if (NOT inclusion_match)
    include_directories(AFTER ${${found_paths}})
    if (DEBUG_OSCD)
      message(STATUS "inclusion append ${${found_paths}} for ${user_set_path}")
    endif()
  endif()
endfunction()

# Boost
# 
# FindBoost.cmake has been included from Cmake's GIT circa the end of 2011
# because most existing installs of cmake had a buggy older version. 
#
# Update this if FindBoost.cmake gets out of sync with the current boost release
# set(Boost_ADDITIONAL_VERSIONS "1.47.0" "1.46.0")

if(DEFINED ENV{VSC_HOME})
    # compiling on a VSC cluster
    set(BOOST_ROOT $ENV{EBROOTBOOST})
endif()

if (WIN32)
  set(Boost_USE_STATIC_LIBS TRUE)
  set(BOOST_STATIC TRUE)
  set(BOOST_THREAD_USE_LIB TRUE)
endif()

if (NOT $ENV{OPENSCAD_LIBRARIES} STREQUAL "")
  set(BOOST_ROOT "$ENV{OPENSCAD_LIBRARIES}")
  if (EXISTS ${BOOST_ROOT}/include/boost)
    # if boost is under OPENSCAD_LIBRARIES, then 
    # don't look in the system paths (workaround FindBoost.cmake bug)
    set(Boost_NO_SYSTEM_PATHS "TRUE")
    message(STATUS "BOOST_ROOT: " ${BOOST_ROOT})
  endif()
endif()

if (NOT $ENV{BOOSTDIR} STREQUAL "")
  set(BOOST_ROOT "$ENV{BOOSTDIR}")
  set(Boost_NO_SYSTEM_PATHS "TRUE")
  set(Boost_DEBUG TRUE)
  message(STATUS "BOOST_ROOT: " ${BOOST_ROOT})
endif()

find_package( Boost 1.35.0 COMPONENTS thread program_options filesystem system regex REQUIRED)
message(STATUS "Boost ${Boost_VERSION} includes found: " ${Boost_INCLUDE_DIRS})
message(STATUS "Boost libraries found:")
foreach(boostlib ${Boost_LIBRARIES})
  message(STATUS "  " ${boostlib})
endforeach()

inclusion(BOOST_ROOT Boost_INCLUDE_DIRS)
if (${Boost_VERSION} LESS 104600)
  add_definitions(-DBOOST_FILESYSTEM_VERSION=3) # Use V3 for boost 1.44-1.45
endif()

# On Mac, we need to link against the correct C++ library. We choose the same one
# as Boost uses.
if(APPLE)
  execute_process(COMMAND grep -q __112basic_string ${Boost_LIBRARIES}
                  RESULT_VARIABLE BOOST_USE_STDLIBCPP)
  if (NOT BOOST_USE_STDLIBCPP)
     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
  endif()
endif()

# Mac OS X
if(APPLE)
  FIND_LIBRARY(COCOA_LIBRARY Cocoa REQUIRED)
  FIND_LIBRARY(APP_SERVICES_LIBRARY ApplicationServices)
endif()


# Eigen

# Turn off Eigen SIMD optimization
if(NOT APPLE)
  if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "^FreeBSD")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DEIGEN_DONT_ALIGN")
  endif()
endif()

# Priority
# 3. EIGENDIR if set
# 1. OPENSCAD_LIBRARIES eigen3
# 4. system's standard include paths for eigen3

set(EIGEN_DIR "$ENV{EIGENDIR}")
set(OPENSCAD_LIBDIR "$ENV{OPENSCAD_LIBRARIES}")

if (EIGEN_DIR)
  set(EIGHINT ${EIGEN_DIR}/include/eigen3 ${EIGEN_DIR})
  find_path(EIGEN_INCLUDE_DIR Eigen/Core HINTS ${EIGHINT})
endif()

if (NOT EIGEN_INCLUDE_DIR)
  find_path(EIGEN_INCLUDE_DIR Eigen/Core HINTS ${OPENSCAD_LIBDIR}/include/eigen3)
endif()

if (NOT EIGEN_INCLUDE_DIR)
  if (${CMAKE_SYSTEM_NAME} MATCHES "^FreeBSD")
    find_path(EIGEN_INCLUDE_DIR Eigen/Core HINTS /usr/local/include/eigen3)
  elseif (${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
    find_path(EIGEN_INCLUDE_DIR Eigen/Core HINTS /usr/pkg/include/eigen3)
  elseif (APPLE)
    find_path(EIGEN_INCLUDE_DIR Eigen/Core HINTS /opt/local/include/eigen3)
  else()
    find_path(EIGEN_INCLUDE_DIR Eigen/Core HINTS /usr/include/eigen3)
  endif()
endif()

if (NOT EIGEN_INCLUDE_DIR)
  message(STATUS "Eigen not found")
else()
  message(STATUS "Eigen found in " ${EIGEN_INCLUDE_DIR})
  inclusion(EIGEN_DIR EIGEN_INCLUDE_DIR)
endif()

###### NULLGL wraps all OpenGL(TM) items (GL, Glew, OpenCSG)
###### Several pages of code fall under this 'if( NOT NULLGL )'
if (NOT NULLGL)

# OpenGL
find_package(OpenGL REQUIRED)
if (NOT OPENGL_GLU_FOUND)
  message(STATUS "GLU not found in system paths...searching $ENV{OPENSCAD_LIBRARIES} ")
  find_library(OPENGL_glu_LIBRARY GLU HINTS $ENV{OPENSCAD_LIBRARIES}/lib)
  if (NOT OPENGL_glu_LIBRARY)
    message(FATAL "GLU library not found")
  endif()
  set(OPENGL_LIBRARIES ${OPENGL_glu_LIBRARY} ${OPENGL_LIBRARIES})
  message(STATUS "OpenGL LIBRARIES: ")
  foreach(GLLIB ${OPENGL_LIBRARIES})
    message(STATUS "  " ${GLLIB})
  endforeach()
endif()

# OpenCSG
if (NOT $ENV{OPENCSGDIR} STREQUAL "")
  set(OPENCSG_DIR "$ENV{OPENCSGDIR}")
elseif (NOT $ENV{OPENSCAD_LIBRARIES} STREQUAL "")
  set(OPENCSG_DIR "$ENV{OPENSCAD_LIBRARIES}")
endif()
if (NOT OPENCSG_INCLUDE_DIR)
  message(STATUS "OPENCSG_DIR: " ${OPENCSG_DIR})
  find_path(OPENCSG_INCLUDE_DIR
            opencsg.h
            HINTS ${OPENCSG_DIR}/include)
  find_library(OPENCSG_LIBRARY
               opencsg
               HINTS ${OPENCSG_DIR}/lib)
  if (NOT OPENCSG_INCLUDE_DIR OR NOT OPENCSG_LIBRARY)
    message(FATAL_ERROR "OpenCSG not found")
  else()
    message(STATUS "OpenCSG include found in " ${OPENCSG_INCLUDE_DIR})
    message(STATUS "OpenCSG library found in " ${OPENCSG_LIBRARY})
  endif()
endif()
inclusion(OPENCSG_DIR OPENCSG_INCLUDE_DIR)

# GLEW

if(WIN32_STATIC_BUILD)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGLEW_STATIC")
endif()

if (NOT $ENV{GLEWDIR} STREQUAL "")
  set(GLEW_DIR "$ENV{GLEWDIR}")
elseif (NOT $ENV{OPENSCAD_LIBRARIES} STREQUAL "")
  set(GLEW_DIR "$ENV{OPENSCAD_LIBRARIES}")
endif()
if (GLEW_DIR)
  find_path(GLEW_INCLUDE_DIR
            GL/glew.h
            HINTS ${GLEW_DIR}/include
	    NO_DEFAULT_PATH)
  find_library(GLEW_LIBRARY
               NAMES GLEW glew
               HINTS ${GLEW_DIR}/lib ${GLEW_DIR}/lib64
	       NO_DEFAULT_PATH)
  if (GLEW_INCLUDE_DIR AND GLEW_LIBRARY)
    set(GLEW_FOUND 1)
  endif()
endif()

if (NOT GLEW_FOUND)
  find_package(GLEW REQUIRED)
endif()

message(STATUS "GLEW include: " ${GLEW_INCLUDE_DIR})
message(STATUS "GLEW library: " ${GLEW_LIBRARY})

inclusion(GLEW_DIR GLEW_INCLUDE_DIR)

endif() ########## NULLGL ENDIF

# Flex/Bison
find_package(BISON REQUIRED)

if(${CMAKE_SYSTEM_NAME} MATCHES "^FreeBSD")
  # FreeBSD has an old flex in /usr/bin and a new flex in /usr/local/bin
  set(FLEX_EXECUTABLE /usr/local/bin/flex)
endif()

# prepend the dir where deps were built
if (NOT $ENV{OPENSCAD_LIBRARIES} STREQUAL "")
  set(OSCAD_DEPS "")
  set(OSCAD_DEPS_PATHS $ENV{OPENSCAD_LIBRARIES}/include)
  inclusion(OSCAD_DEPS OSCAD_DEPS_PATHS)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
  include_directories( /usr/pkg/include /usr/X11R7/include )
  set(FLEX_EXECUTABLE /usr/pkg/bin/flex)
  if(NOT ${CMAKE_CXX_COMPILER} MATCHES ".*clang.*")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
  endif()
endif()

find_package(FLEX REQUIRED)
# The COMPILE_FLAGS and forced C++ compiler is just to be compatible with qmake
if (WIN32)
  set(FLEX_UNISTD_FLAG "-DYY_NO_UNISTD_H")
endif()
FLEX_TARGET(OpenSCADlexer ../src/lexer.l ${CMAKE_CURRENT_BINARY_DIR}/lexer.cpp COMPILE_FLAGS "-Plexer ${FLEX_UNISTD_FLAG}")
BISON_TARGET(OpenSCADparser ../src/parser.y ${CMAKE_CURRENT_BINARY_DIR}/parser_yacc.c COMPILE_FLAGS "-p parser")
ADD_FLEX_BISON_DEPENDENCY(OpenSCADlexer OpenSCADparser)
set_source_files_properties(${CMAKE_CURRENT_BINARY_DIR}/parser_yacc.c PROPERTIES LANGUAGE "CXX")

# CGAL

# Disable rounding math check to allow usage of Valgrind
# This is needed as Valgrind currently does not correctly
# handle rounding modes used by CGAL.
# set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCGAL_DISABLE_ROUNDING_MATH_CHECK=ON")

if (NOT $ENV{CGALDIR} STREQUAL "")
  set(CGAL_DIR "$ENV{CGALDIR}")
elseif (NOT $ENV{OPENSCAD_LIBRARIES} STREQUAL "")
  if (EXISTS "$ENV{OPENSCAD_LIBRARIES}/lib/CGAL")
    set(CGAL_DIR "$ENV{OPENSCAD_LIBRARIES}/lib/CGAL")
    set(CMAKE_MODULE_PATH "${CGAL_DIR}" ${CMAKE_MODULE_PATH})
  elseif (EXISTS "$ENV{OPENSCAD_LIBRARIES}/include/CGAL")
    set(CGAL_DIR "$ENV{OPENSCAD_LIBRARIES}")
    set(CMAKE_MODULE_PATH "${CGAL_DIR}" ${CMAKE_MODULE_PATH})
  endif()
endif()
message(STATUS "CGAL_DIR: " ${CGAL_DIR})
find_package(CGAL REQUIRED)
message(STATUS "CGAL config found in " ${CGAL_USE_FILE} )
foreach(cgal_incdir ${CGAL_INCLUDE_DIRS})
  message(STATUS "CGAL include found in " ${cgal_incdir} )
endforeach()
message(STATUS "CGAL libraries found in " ${CGAL_LIBRARIES_DIR} )
if("${CGAL_MAJOR_VERSION}.${CGAL_MINOR_VERSION}" VERSION_LESS 3.6)
  message(FATAL_ERROR "CGAL >= 3.6 required")
endif()
inclusion(CGAL_DIR CGAL_INCLUDE_DIRS)

#Remove bad BOOST libraries from CGAL 3rd party dependencies when they don't exist (such as on 64-bit Ubuntu 13.10).
#Libs of concern are /usr/lib/libboost_thread.so;/usr/lib/libboost_system.so;
#Confirmed bug in CGAL @ https://bugs.launchpad.net/ubuntu/+source/cgal/+bug/1242111
string(FIND "${CGAL_3RD_PARTY_LIBRARIES}" "/usr/lib/libboost_system.so" FIND_POSITION  )
if(NOT "-1" STREQUAL ${FIND_POSITION} )
  if(NOT EXISTS "/usr/lib/libboost_system.so")
    MESSAGE( STATUS "CGAL_3RD_PARTY_LIBRARIES:Removing non-existent /usr/lib/libboost_system.so" )
    string(REPLACE "/usr/lib/libboost_system.so" "" CGAL_3RD_PARTY_LIBRARIES ${CGAL_3RD_PARTY_LIBRARIES})
  endif()
endif() 
string(FIND "${CGAL_3RD_PARTY_LIBRARIES}" "/usr/lib/libboost_thread.so" FIND_POSITION  )
if(NOT "-1" STREQUAL ${FIND_POSITION} )
  if(NOT EXISTS "/usr/lib/libboost_thread.so")
    MESSAGE( STATUS "CGAL_3RD_PARTY_LIBRARIES:Removing non-existent /usr/lib/libboost_thread.so" )
    string(REPLACE "/usr/lib/libboost_thread.so" "" CGAL_3RD_PARTY_LIBRARIES ${CGAL_3RD_PARTY_LIBRARIES})
  endif()
endif() 

if (${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
  foreach(CGAL3RDPLIB ${CGAL_3RD_PARTY_LIBRARIES})
    if(NOT EXISTS "${CGAL3RDPLIB}")
      MESSAGE( STATUS " Removing non-existent ${CGAL3RDPLIB}" )
      string(REPLACE "${CGAL3RDPLIB}" "" CGAL_3RD_PARTY_LIBRARIES ${CGAL_3RD_PARTY_LIBRARIES})
    endif()
  endforeach()
endif()

MESSAGE(STATUS "CGAL 3RD PARTY LIBS:")
foreach(CGAL3RDPLIB ${CGAL_3RD_PARTY_LIBRARIES})
  MESSAGE(STATUS " ${CGAL3RDPLIB}" )
endforeach()

if(${CMAKE_CXX_COMPILER} MATCHES ".*clang.*" AND NOT ${CGAL_CXX_FLAGS_INIT} STREQUAL "" )
	string(REPLACE "-frounding-math" "" CGAL_CXX_FLAGS_INIT ${CGAL_CXX_FLAGS_INIT})
	string(REPLACE "--param=ssp-buffer-size=4" "" CGAL_CXX_FLAGS_INIT ${CGAL_CXX_FLAGS_INIT})
endif()

# GLib2

find_package(GLIB2 2.2.0 REQUIRED)
add_definitions(${GLIB2_DEFINITIONS})
inclusion(GLIB2_DIR GLIB2_INCLUDE_DIRS)

# find libraries using pkg-config
find_package(PkgConfig REQUIRED)
include(PkgConfigTools.cmake)
save_pkg_config_env()

if (DEFINED ENV{OPENSCAD_LIBRARIES})
  set(ENV{PKG_CONFIG_PATH} "$ENV{OPENSCAD_LIBRARIES}/lib/pkgconfig")
endif()

pkg_check_modules(FONTCONFIG REQUIRED fontconfig>=2.8.0)
if (FONTCONFIG_VERSION)
  message(STATUS "fontconfig ${FONTCONFIG_VERSION} found: ${FONTCONFIG_INCLUDE_DIRS}")
endif()

pkg_check_modules(FREETYPE REQUIRED freetype2>=2.4.9)
if (FREETYPE_VERSION)
  message(STATUS "freetype2 ${FREETYPE_VERSION} found: ${FREETYPE_INCLUDE_DIRS}")
endif()

if (${NOHARFBUZZ} MATCHES "OFF")
    pkg_check_modules(HARFBUZZ REQUIRED harfbuzz>=0.9.19)
    if (HARFBUZZ_VERSION)
    message(STATUS "harfbuzz ${HARFBUZZ_VERSION} found: ${HARFBUZZ_INCLUDE_DIRS}")
    endif()
else()
    message(STATUS "harfbuzz not used")
endif()

restore_pkg_config_env()

add_definitions(${FONTCONFIG_CFLAGS})
add_definitions(${FREETYPE_CFLAGS})
add_definitions(${HARFBUZZ_CFLAGS})

# Image comparison - expected test image vs actual generated image

if (DIFFPNG)
  # How to set cflags to optimize the executable?
  set(IMAGE_COMPARE_EXECUTABLE ${CMAKE_CURRENT_BINARY_DIR}/diffpng)
  set(COMPARATOR "diffpng")
  add_executable(diffpng diffpng.cpp ../src/lodepng.cpp)
  set(SKIP_IMAGEMAGICK 1)
  message(STATUS "using diffpng for image comparison")
endif()

if (SKIP_IMAGEMAGICK)
  if (NOT DIFFPNG)
    # cross-building depends on this
    set(IMAGE_COMPARE_EXECUTABLE "/bin/echo")
  endif()
else()
  find_package(ImageMagick COMPONENTS convert)
  if (ImageMagick_convert_FOUND)
    message(STATUS "ImageMagick convert executable found: " ${ImageMagick_convert_EXECUTABLE})
    set(IMAGE_COMPARE_EXECUTABLE ${ImageMagick_convert_EXECUTABLE})
  else()
    message(STATUS "Couldn't find imagemagick 'convert' program")
    set(DIFFPNG 1)
  endif()
  if ( "${ImageMagick_VERSION_STRING}" VERSION_LESS "6.5.9.4" )
    message(STATUS "ImageMagick version less than 6.5.9.4, cannot use -morphology comparison")
    message(STATUS "ImageMagick Using older image comparison method")
    set(COMPARATOR "old")
  endif()

  execute_process(COMMAND ${IMAGE_COMPARE_EXECUTABLE} --version OUTPUT_VARIABLE IM_OUT )
  if ( ${IM_OUT} MATCHES "OpenMP" )
    # http://www.daniloaz.com/en/617/systems/high-cpu-load-when-converting-images-with-imagemagick
    message(STATUS "ImageMagick: OpenMP bug workaround - setting MAGICK_THREAD_LIMIT=1")
    set(CTEST_ENVIRONMENT "${CTEST_ENVIRONMENT};MAGICK_THREAD_LIMIT=1")
  endif()

  message(STATUS "Comparing magicktest1.png with magicktest2.png")
  set(IM_TEST_FILES "${CMAKE_CURRENT_SOURCE_DIR}/magicktest1.png" "${CMAKE_CURRENT_SOURCE_DIR}/magicktest2.png")
  set(COMPARE_ARGS ${IMAGE_COMPARE_EXECUTABLE} ${IM_TEST_FILES} -alpha Off -compose difference -composite -threshold 10% -morphology Erode Square -format %[fx:w*h*mean] info:)
  # compare arguments taken from test_cmdline_tool.py
  message(STATUS "Running ImageMagick compare: ${COMPARE_ARGS}")
  execute_process(COMMAND ${COMPARE_ARGS} RESULT_VARIABLE IM_RESULT OUTPUT_VARIABLE IM_OUT )
  message(STATUS "Result: ${IM_RESULT}")
  if ( NOT ${IM_RESULT} STREQUAL "0" )
    message(STATUS "magicktest1.png and magicktest2.png were incorrectly detected as identical")
    message(STATUS "Using alternative image comparison")
    set(DIFFPNG 1)
  endif()
endif()

# Internal includes
include_directories(../src)

# Handle OpenSCAD version based on VERSION env. variable.
# Use current timestamp if not specified (development builds)
if ("$ENV{VERSION}" STREQUAL "")
  # Timestamp is only available in cmake >= 2.8.11  
  if("${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}" VERSION_GREATER 2.8.10)
    string(TIMESTAMP VERSION "%Y.%m.%d")
  else()
    set(VERSION "2013.06")
  endif()
else()
  set(VERSION $ENV{VERSION})
endif()
message(STATUS "OpenSCAD version: ${VERSION}")
string(REGEX MATCHALL "^[0-9]+|[0-9]+|[0-9]+$" MYLIST "${VERSION}")
list(GET MYLIST 0 OPENSCAD_YEAR)
list(GET MYLIST 1 OPENSCAD_MONTH)
math(EXPR OPENSCAD_MONTH ${OPENSCAD_MONTH}) # get rid of leading zero
list(LENGTH MYLIST VERSIONLEN)
if (${VERSIONLEN} EQUAL 3)
  list(GET MYLIST 2 OPENSCAD_DAY)
  math(EXPR OPENSCAD_DAY ${OPENSCAD_DAY}) # get rid of leading zero
endif()

add_definitions(-DOPENSCAD_VERSION=${VERSION} -DOPENSCAD_YEAR=${OPENSCAD_YEAR} -DOPENSCAD_MONTH=${OPENSCAD_MONTH})
if (DEFINED OPENSCAD_DAY)
  add_definitions(-DOPENSCAD_DAY=${OPENSCAD_DAY})
endif()

add_definitions(-DOPENSCAD_TESTING -DENABLE_EXPERIMENTAL)

# Search for MCAD in correct place
set(CTEST_ENVIRONMENT "${CTEST_ENVIRONMENT};OPENSCADPATH=${CMAKE_CURRENT_SOURCE_DIR}/../libraries")

# Platform specific settings

if(APPLE)
	message(STATUS "Offscreen OpenGL Context - using Apple CGL")
	set(OFFSCREEN_CTX_SOURCE "OffscreenContextCGL.mm" CACHE TYPE STRING)
	set(OFFSCREEN_IMGUTILS_SOURCE "imageutils-macosx.cc" CACHE TYPE STRING)
	set(PLATFORMUTILS_SOURCE "PlatformUtils-mac.mm" CACHE TYPE STRING)
elseif(UNIX)
	message(STATUS "Offscreen OpenGL Context - using Unix GLX")
	set(OFFSCREEN_CTX_SOURCE "OffscreenContextGLX.cc" CACHE TYPE STRING)
	set(OFFSCREEN_IMGUTILS_SOURCE "imageutils-lodepng.cc" CACHE TYPE STRING)
	set(PLATFORMUTILS_SOURCE "PlatformUtils-posix.cc" CACHE TYPE STRING)
elseif(WIN32)
	message(STATUS "Offscreen OpenGL Context - using Microsoft WGL")
	set(OFFSCREEN_CTX_SOURCE "OffscreenContextWGL.cc" CACHE TYPE STRING)
	set(OFFSCREEN_IMGUTILS_SOURCE "imageutils-lodepng.cc" CACHE TYPE STRING)
	set(PLATFORMUTILS_SOURCE "PlatformUtils-win.cc" CACHE TYPE STRING)
endif()

set(CORE_SOURCES
  tests-common.cc 
  headlessconfig.h.in
  ../src/parsersettings.cc
  ../src/mathc99.cc
  ../src/linalg.cc
  ../src/colormap.cc
  ../src/Camera.cc
  ../src/handle_dep.cc 
  ../src/value.cc 
  ../src/calc.cc 
  ../src/expr.cc 
  ../src/func.cc 
  ../src/localscope.cc 
  ../src/module.cc 
  ../src/ModuleCache.cc 
  ../src/node.cc 
  ../src/context.cc 
  ../src/modcontext.cc 
  ../src/evalcontext.cc 
  ../src/feature.cc
  ../src/csgterm.cc 
  ../src/csgtermnormalizer.cc 
  ../src/Geometry.cc 
  ../src/Polygon2d.cc 
  ../src/polyset.cc 
  ../src/csgops.cc 
  ../src/transform.cc 
  ../src/color.cc 
  ../src/primitives.cc 
  ../src/projection.cc 
  ../src/cgaladv.cc 
  ../src/surface.cc 
  ../src/control.cc 
  ../src/render.cc 
  ../src/rendersettings.cc 
  ../src/dxfdata.cc 
  ../src/dxfdim.cc 
  ../src/offset.cc 
  ../src/linearextrude.cc 
  ../src/rotateextrude.cc 
  ../src/text.cc 
  ../src/printutils.cc 
  ../src/fileutils.cc 
  ../src/progress.cc 
  ../src/boost-utils.cc 
  ../src/FontCache.cc
  ../src/DrawingCallback.cc
  ../src/FreetypeRenderer.cc
  ../src/lodepng.cpp
  ../src/PlatformUtils.cc 
  ../src/${PLATFORMUTILS_SOURCE}
  ${FLEX_OpenSCADlexer_OUTPUTS}
  ${BISON_OpenSCADparser_OUTPUTS})

set(NOCGAL_SOURCES
  ../src/builtin.cc 
  ../src/import.cc
  ../src/export.cc
  ../src/LibraryInfo.cc)

set(CGAL_SOURCES
  ${NOCGAL_SOURCES}
  ../src/CSGTermEvaluator.cc 
  ../src/CGAL_Nef_polyhedron.cc 
  ../src/cgalutils.cc 
  ../src/cgalutils-tess.cc 
  ../src/CGALCache.cc
  ../src/CGAL_Nef_polyhedron_DxfData.cc
  ../src/Polygon2d-CGAL.cc
  ../src/polyset-utils.cc 
  ../src/svg.cc
  ../src/GeometryEvaluator.cc)

set(COMMON_SOURCES
  ../src/nodedumper.cc 
  ../src/traverser.cc 
  ../src/GeometryCache.cc 
  ../src/clipper-utils.cc 
  ../src/polyclipping/clipper.cpp
  ../src/Tree.cc)

#
# Offscreen OpenGL context source code
#

set(OFFSCREEN_SOURCES
  ../src/GLView.cc
  ../src/OffscreenView.cc
  ../src/${OFFSCREEN_CTX_SOURCE}
  ../src/${OFFSCREEN_IMGUTILS_SOURCE}
  ../src/imageutils.cc
  ../src/fbo.cc
  ../src/system-gl.cc
  ../src/export_png.cc
  ../src/CGALRenderer.cc
  ../src/ThrownTogetherRenderer.cc
  ../src/renderer.cc
  ../src/render.cc
  ../src/OpenCSGRenderer.cc
)

if(NULLGL)
  message(STATUS "NULLGL is set. Overriding previous OpenGL(TM) settings")
  set(OFFSCREEN_SOURCES
    ../src/GLView.h
    ../src/NULLGL.cc # contains several 'nullified' versions of above .cc files
    ../src/OffscreenView.cc
    ../src/OffscreenContextNULL.cc
    ../src/export_png.cc
    ../src/${OFFSCREEN_IMGUTILS_SOURCE}
    ../src/imageutils.cc
    ../src/renderer.cc
    ../src/render.cc)
endif()


#make cmake options available in code (delete build dir on changes!)
CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/headlessconfig.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/headlessconfig.h)

add_library(tests-core STATIC ${CORE_SOURCES})
target_link_libraries(tests-core ${OPENGL_LIBRARIES} ${GLIB2_LIBRARIES} ${FONTCONFIG_LDFLAGS} ${FREETYPE_LDFLAGS} ${HARFBUZZ_LDFLAGS} ${Boost_LIBRARIES} ${COCOA_LIBRARY})

add_library(tests-common STATIC ${COMMON_SOURCES})
target_link_libraries(tests-common tests-core)

add_library(tests-cgal STATIC ${CGAL_SOURCES})
set_target_properties(tests-cgal PROPERTIES COMPILE_FLAGS "${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")
target_link_libraries(tests-cgal tests-common ${CGAL_LIBRARY} ${CGAL_3RD_PARTY_LIBRARIES} ${GMP_LIBRARIES} ${MPFR_LIBRARIES})

#
# Create non-CGAL tests
#
if (NOT NULLGL)
  add_library(tests-nocgal STATIC ${NOCGAL_SOURCES})
  set_target_properties(tests-nocgal PROPERTIES COMPILE_FLAGS ${ENABLE_OPENCSG_FLAG})
  target_link_libraries(tests-nocgal tests-common)
else()
  message(STATUS "NULLGL: cannot use GL/GLU tessellator. see dxftess.cc")
  message(STATUS "NULLGL: non-CGAL tests will use CGAL's tessellator")
  add_library(tests-nocgal STATIC ${NOCGAL_SOURCES})
  set_target_properties(tests-nocgal PROPERTIES COMPILE_FLAGS "${ENABLE_OPENCSG_FLAG}")
  target_link_libraries(tests-nocgal tests-common)
endif()

add_library(tests-offscreen STATIC ${OFFSCREEN_SOURCES})
set_target_properties(tests-offscreen PROPERTIES COMPILE_FLAGS "${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")

#
# modulecachetest
#
##add_executable(modulecachetest modulecachetest.cc)
##target_link_libraries(modulecachetest tests-nocgal)

#
# csgtexttest
#
##add_executable(csgtexttest csgtexttest.cc CSGTextRenderer.cc CSGTextCache.cc)
##target_link_libraries(csgtexttest tests-nocgal)

#
# cgalcachetest
#
##add_executable(cgalcachetest cgalcachetest.cc)
##set_target_properties(cgalcachetest PROPERTIES COMPILE_FLAGS "${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")
##target_link_libraries(cgalcachetest tests-cgal ${GLEW_LIBRARY})

#
# openscad no-qt
#
message(STATUS "creating nogui executable target")
add_executable(openscad_nogui ../src/openscad.cc)
set_target_properties(openscad_nogui PROPERTIES COMPILE_FLAGS "-fno-strict-aliasing -DEIGEN_DONT_ALIGN ${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")
target_link_libraries(openscad_nogui tests-offscreen tests-cgal ${GLEW_LIBRARY} ${OPENCSG_LIBRARY} ${APP_SERVICES_LIBRARY})

message(STATUS "CPPFLAGS: ${CMAKE_CXX_FLAGS}")
