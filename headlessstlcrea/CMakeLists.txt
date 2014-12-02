# instructions - see ../doc/testing.txt

set(DEBUG_OSCD 1) # print debug info during cmake

SET(CMAKE_BUILD_TYPE Debug)
#SET(CMAKE_BUILD_TYPE Release)

project(openscad_nogui)


# User settable Target options (remove build directory on changes!)
option (NOTEXTOPTION      "Compile without text()"      ON)
option (NOMCAD            "Compile without MCAD"        ON)


if (${NOTEXTOPTION} MATCHES "ON")
    SET(NOTEXT 1)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNOTEXT")
    message(STATUS "text() Disabled")
endif()

if (${NOMCAD} MATCHES "ON")
    SET(NOMCAD 1)
endif()

cmake_minimum_required(VERSION 2.8)
if("${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}" VERSION_GREATER 2.8.3)
  # Explicitly use new include policy to avoid globally shadowing included modules
  # http://www.cmake.org/cmake/help/cmake-2-8-docs.html#policy:CMP0017
  cmake_policy(SET CMP0017 NEW)
endif()

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}" ${CMAKE_MODULE_PATH})
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/../tests" ${CMAKE_MODULE_PATH})

# Needed for cmake < 2.8.3
include(../tests/CMakeParseArguments.cmake)

# NULLGL is set by default
#- Allow us to build without OpenGL(TM). run 'cmake .. -DNULLGL=1'
set(NULLGL 1)

include(../tests/OpenscadDeps.cmake)

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
# openscad no-qt
#
message(STATUS "creating nogui executable target")
add_executable(openscad_nogui ../src/openscad.cc)
set_target_properties(openscad_nogui PROPERTIES COMPILE_FLAGS "-fno-strict-aliasing -DEIGEN_DONT_ALIGN ${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")
target_link_libraries(openscad_nogui tests-offscreen tests-cgal ${GLEW_LIBRARY} ${OPENCSG_LIBRARY} ${APP_SERVICES_LIBRARY})

message(STATUS "CPPFLAGS: ${CMAKE_CXX_FLAGS}")
