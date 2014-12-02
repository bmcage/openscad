# instructions - see ../doc/testing.txt

#set(DEBUG_OSCD 1) # print debug info during cmake

cmake_minimum_required(VERSION 2.8)
if("${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}" VERSION_GREATER 2.8.3)
  # Explicitly use new include policy to avoid globally shadowing included modules
  # http://www.cmake.org/cmake/help/cmake-2-8-docs.html#policy:CMP0017
  cmake_policy(SET CMP0017 NEW)
endif()


set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}")

# Needed for cmake < 2.8.3
include(CMakeParseArguments.cmake)

project(tests)

#no MCAD (MCAD tests will fail!)
#SET(NOMCAD 1)

include(OpenscadDeps.cmake)

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
  set_target_properties(tests-nocgal PROPERTIES COMPILE_FLAGS ${ENABLE_OPENCSG_FLAG})
  target_link_libraries(tests-nocgal tests-common)
endif()

add_library(tests-offscreen STATIC ${OFFSCREEN_SOURCES})
set_target_properties(tests-offscreen PROPERTIES COMPILE_FLAGS "${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")

#
# modulecachetest
#
add_executable(modulecachetest modulecachetest.cc)
target_link_libraries(modulecachetest tests-nocgal)

#
# csgtexttest
#
add_executable(csgtexttest csgtexttest.cc CSGTextRenderer.cc CSGTextCache.cc)
target_link_libraries(csgtexttest tests-nocgal)

#
# cgalcachetest
#
add_executable(cgalcachetest cgalcachetest.cc)
set_target_properties(cgalcachetest PROPERTIES COMPILE_FLAGS "${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")
target_link_libraries(cgalcachetest tests-cgal ${GLEW_LIBRARY})

#
# openscad no-qt
#
add_executable(openscad_nogui ../src/openscad.cc)
set_target_properties(openscad_nogui PROPERTIES COMPILE_FLAGS "-fno-strict-aliasing -DEIGEN_DONT_ALIGN ${ENABLE_OPENCSG_FLAG} -DENABLE_CGAL ${CGAL_CXX_FLAGS_INIT}")
target_link_libraries(openscad_nogui tests-offscreen tests-cgal ${GLEW_LIBRARY} ${OPENCSG_LIBRARY} ${APP_SERVICES_LIBRARY})

#
# GUI binary tests
#
#if(APPLE)
#  set(OPENSCAD_BINPATH "${CMAKE_CURRENT_SOURCE_DIR}/../OpenSCAD.app/Contents/MacOS/OpenSCAD")
#elseif (MINGW_CROSS_ENV_DIR) 
#  set(OPENSCAD_BINPATH "${CMAKE_CURRENT_SOURCE_DIR}/../mingw32/release/openscad.exe")
#elseif(WIN32)
#  set(OPENSCAD_BINPATH "${CMAKE_CURRENT_SOURCE_DIR}/../Release/openscad.exe")
#else()
#  set(OPENSCAD_BINPATH "${CMAKE_CURRENT_SOURCE_DIR}/../openscad")
#endif()

#if(EXISTS "${CMAKE_CURRENT_BINARY_DIR}/openscad")
#  set(OPENSCAD_BINPATH "${CMAKE_CURRENT_BINARY_DIR}/openscad")
#endif()

#if(EXISTS "${OPENSCAD_BINPATH}")
#  message(STATUS "Found OpenSCAD binary: ${OPENSCAD_BINPATH}")
#else()
#  message(STATUS "Couldn't find the OpenSCAD binary: ${OPENSCAD_BINPATH}")
#  message(FATAL_ERROR "Please build the OpenSCAD binary and place it here: ${OPENSCAD_BINPATH}" )
#endif()

if(WIN32)
  set(OPENSCAD_BINPATH "${CMAKE_CURRENT_BINARY_DIR}/openscad_nogui.exe")
else()
  set(OPENSCAD_BINPATH "${CMAKE_CURRENT_BINARY_DIR}/openscad_nogui")
endif()

#
# Tags tests as disabled. This is more convenient than removing them manually
# from the lists of filenames
#
macro(disable_tests)
  foreach (TESTNAME ${ARGN})
#    message("Disabling ${TESTNAME}")
    list(APPEND DISABLED_TESTS ${TESTNAME})
  endforeach()
endmacro()

#
# Tags the given tests as belonging to the given CONFIG, i.e. will
# only be executed when run using ctest -C <CONFIG>
#
# Usage example: set_test_config(Heavy dumptest_testname opencsgtest_testname2)
#
function(set_test_config CONFIG)
  list(APPEND ${CONFIG}_TEST_CONFIG ${ARGN})
  list(FIND TEST_CONFIGS ${CONFIG} FOUND)
  if (FOUND EQUAL -1)
    list(APPEND TEST_CONFIGS ${CONFIG})
    # Export to parent scope
    set(TEST_CONFIGS ${TEST_CONFIGS} PARENT_SCOPE)
  endif()
  # Export to parent scope
  set(${CONFIG}_TEST_CONFIG ${${CONFIG}_TEST_CONFIG} PARENT_SCOPE)
endfunction()

#
# Returns a list of test configs 
#
function(get_test_config TESTNAME CONFIGS)
  foreach(CONFIG ${TEST_CONFIGS})
    list(FIND ${CONFIG}_TEST_CONFIG ${TESTNAME} IDX)
    if (${IDX} GREATER -1)
      list(APPEND ${CONFIGS} ${CONFIG})
    endif()
  endforeach()
  if (${CONFIGS})
    # Convert to a format understood by add_test()
    string(REPLACE ";" "|" ${${CONFIGS}} ${CONFIGS})
    # Export to parent scope
    set(${CONFIGS} ${${CONFIGS}} PARENT_SCOPE)
  endif()
endfunction()

#
# Returns into the FULLNAME variable the global full test name (identifier) 
# given a test command and source filename
#
function(get_test_fullname TESTCMD FILENAME FULLNAME)
  get_filename_component(TESTCMD_NAME ${TESTCMD} NAME_WE)
  get_filename_component(TESTNAME ${FILENAME} NAME_WE)
  string(REPLACE " " "_" TESTNAME ${TESTNAME}) # Test names cannot include spaces
  set(${FULLNAME} ${TESTCMD_NAME}_${TESTNAME})
  # Export to parent scope
  set(${FULLNAME} ${${FULLNAME}} PARENT_SCOPE)
endfunction()

#
# Check if a test file is a 2D test
#
function(is_2d FULLNAME RESULT)
  list(FIND ALL_2D_FILES ${FULLNAME} IDX)
  if (${IDX} GREATER -1)
    set(${RESULT} 1 PARENT_SCOPE)
  else()
    set(${RESULT} PARENT_SCOPE)
  endif()
endfunction()

#
# This functions adds cmd-line tests given files.
#
# Usage add_cmdline_test(testbasename [EXE <executable>] [ARGS <args to exe>]
#                        [SCRIPT <script>]
#                        [EXPECTEDDIR <shared dir>] SUFFIX <suffix> FILES <test files>)
#
find_package(PythonInterp)
function(add_cmdline_test TESTCMD_BASENAME)
  cmake_parse_arguments(TESTCMD "" "EXE;SCRIPT;SUFFIX;EXPECTEDDIR" "FILES;ARGS" ${ARGN})

  # If sharing results with another test, pass on this to the python script
  if (TESTCMD_EXPECTEDDIR)
    set(EXTRA_OPTIONS -e ${TESTCMD_EXPECTEDDIR})
  endif()

  if (TESTCMD_EXE)
    set(TESTNAME_OPTION -t ${TESTCMD_BASENAME})
  else()
    # If no executable was specified, assume it was built by us and resides here
    set(TESTCMD_EXE ${CMAKE_BINARY_DIR}/${TESTCMD_BASENAME})
  endif()

  # Add tests from args
  foreach (SCADFILE ${TESTCMD_FILES})
    get_filename_component(FILE_BASENAME ${SCADFILE} NAME_WE)
    string(REPLACE " " "_" FILE_BASENAME ${FILE_BASENAME}) # Test names cannot include spaces
    set(TEST_FULLNAME "${TESTCMD_BASENAME}_${FILE_BASENAME}")
    list(FIND DISABLED_TESTS ${TEST_FULLNAME} DISABLED)

    if (${DISABLED} EQUAL -1)

      set(EXPERIMENTAL_OPTION "")
      string(REGEX MATCH "csgtexttest" match_result1 ${TEST_FULLNAME})
      string(REGEX MATCH "cgalstlsanity" match_result2 ${TEST_FULLNAME})
      string(REGEX MATCH "dumptest" match_result3 ${TEST_FULLNAME})
      if( "${match_result1}" STREQUAL "" )
       if( "${match_result2}" STREQUAL "" )
        if( "${match_result3}" STREQUAL "" )
         set(EXPERIMENTAL_OPTION "--enable=text")
        endif()
       endif()
      endif()

      # 2D tests should be viewed from the top, not an angle.
      set(CAMERA_OPTION "")
      is_2d(${SCADFILE} IS2D)
      if (IS2D)
        set(CAMERA_OPTION "--camera=0,0,100,0,0,0" "--viewall" "--autocenter" "--projection=ortho")
      endif()

      # Handle configurations
      unset(FOUNDCONFIGS)
      get_test_config(${TEST_FULLNAME} FOUNDCONFIGS)
      if (NOT FOUNDCONFIGS)
        set_test_config(Default ${TEST_FULLNAME})
      endif()
      set_test_config(All ${TEST_FULLNAME})
      unset(FOUNDCONFIGS)
      get_test_config(${TEST_FULLNAME} FOUNDCONFIGS)
      set(CONFARG CONFIGURATIONS)
      set(CONFVAL ${FOUNDCONFIGS})

      # The python script cannot extract the testname when given extra parameters
      if (TESTCMD_ARGS)
        set(FILENAME_OPTION -f ${FILE_BASENAME})
      endif()

      add_test(NAME ${TEST_FULLNAME} ${CONFARG} ${CONFVAL} COMMAND ${PYTHON_EXECUTABLE} ${tests_SOURCE_DIR}/test_cmdline_tool.py --comparator=${COMPARATOR} -c ${IMAGE_COMPARE_EXECUTABLE} -s ${TESTCMD_SUFFIX} ${EXTRA_OPTIONS} ${TESTNAME_OPTION} ${FILENAME_OPTION} ${TESTCMD_EXE} ${TESTCMD_SCRIPT} "${SCADFILE}" ${CAMERA_OPTION} ${EXPERIMENTAL_OPTION} ${TESTCMD_ARGS})
      set_property(TEST ${TEST_FULLNAME} PROPERTY ENVIRONMENT "${CTEST_ENVIRONMENT}")
    endif()
  endforeach()
endfunction()

#
# Usage add_failing_test(testbasename RETVAL <expected return value>
#                        [EXE <executable>] [SCRIPT <script>] [ARGS <args to exe>]
#                        FILES <test files>)
#
function(add_failing_test TESTCMD_BASENAME)
  cmake_parse_arguments(TESTCMD "" "RETVAL;EXE;SCRIPT;" "FILES;ARGS" ${ARGN})

  if (TESTCMD_EXE)
    set(TESTNAME_OPTION -t ${TESTCMD_BASENAME})
  else()
    # If no executable was specified, assume it was built by us and resides here
    set(TESTCMD_EXE ${CMAKE_BINARY_DIR}/${TESTCMD_BASENAME})
  endif()

  # Add tests from args
  foreach (SCADFILE ${TESTCMD_FILES})
    get_filename_component(FILE_BASENAME ${SCADFILE} NAME_WE)
    string(REPLACE " " "_" FILE_BASENAME ${FILE_BASENAME}) # Test names cannot include spaces
    set(TEST_FULLNAME "${TESTCMD_BASENAME}_${FILE_BASENAME}")
    list(FIND DISABLED_TESTS ${TEST_FULLNAME} DISABLED)

    if (${DISABLED} EQUAL -1)
      # Handle configurations
      unset(FOUNDCONFIGS)
      get_test_config(${TEST_FULLNAME} FOUNDCONFIGS)
      if (NOT FOUNDCONFIGS)
        set_test_config(Default ${TEST_FULLNAME})
      endif()
      set_test_config(All ${TEST_FULLNAME})
      unset(FOUNDCONFIGS)
      get_test_config(${TEST_FULLNAME} FOUNDCONFIGS)
      set(CONFARG CONFIGURATIONS)
      set(CONFVAL ${FOUNDCONFIGS})

      # The python script cannot extract the testname when given extra parameters
      if (TESTCMD_ARGS)
        set(FILENAME_OPTION -f ${FILE_BASENAME})
      endif()

      add_test(NAME ${TEST_FULLNAME} ${CONFARG} ${CONFVAL} COMMAND ${TESTCMD_EXE} ${TESTCMD_SCRIPT} "${SCADFILE}" ${TESTCMD_ARGS})
      set_property(TEST ${TEST_FULLNAME} PROPERTY ENVIRONMENT "${CTEST_ENVIRONMENT}")
    endif()
  endforeach()
endfunction()

enable_testing()



set_directory_properties(PROPERTIES TEST_INCLUDE_FILE "${CMAKE_SOURCE_DIR}/EnforceConfig.cmake")

# Subst files
configure_file(${CMAKE_SOURCE_DIR}/../testdata/scad/templates/include-tests-template.scad
               ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/include-tests.scad)
configure_file(${CMAKE_SOURCE_DIR}/../testdata/scad/templates/use-tests-template.scad
               ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/use-tests.scad)
configure_file(${CMAKE_SOURCE_DIR}/../testdata/scad/templates/import_stl-tests-template.scad
               ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/import_stl-tests.scad)
configure_file(${CMAKE_SOURCE_DIR}/../testdata/scad/templates/import_dxf-tests-template.scad
               ${CMAKE_SOURCE_DIR}/../testdata/scad/2D/features/import_dxf-tests.scad)

# Find all scad files
file(GLOB FEATURES_3D_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/*.scad)
file(GLOB FEATURES_2D_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/2D/features/*.scad)
file(GLOB DEPRECATED_3D_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/deprecated/*.scad)
file(GLOB ISSUES_3D_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/issues/*.scad)
file(GLOB SCAD_DXF_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/dxf/*.scad)
file(GLOB FUNCTION_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/functions/*.scad)
file(GLOB_RECURSE EXAMPLE_3D_FILES ${CMAKE_SOURCE_DIR}/../examples/*.scad)

list(REMOVE_ITEM EXAMPLE_3D_FILES ${CMAKE_SOURCE_DIR}/../examples/Shapes/flat_body.scad)
list(APPEND EXAMPLE_2D_FILES ${CMAKE_SOURCE_DIR}/../examples/Shapes/flat_body.scad)

list(APPEND EXAMPLE_FILES ${EXAMPLE_3D_FILES} ${EXAMPLE_2D_FILES})

list(APPEND ECHO_FILES ${FUNCTION_FILES}
            ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/for-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/echo-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/escape-test.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/parser-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/builtin-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/dim-all.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/string-test.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/string-indexing.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/string-unicode.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/chr-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/vector-values.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/search-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/search-tests-unicode.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/recursion-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/value-reassignment-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/value-reassignment-tests2.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/variable-scope-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/scope-assignment-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/lookup-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/expression-shortcircuit-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/parent_module-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/children-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/range-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/no-break-space-test.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/unicode-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/utf8-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/concat-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/include-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/include-recursive-test.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/operators-tests.scad
            ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/empty-stl.scad)

list(APPEND DUMPTEST_FILES ${FEATURES_2D_FILES} ${FEATURES_3D_FILES} ${DEPRECATED_3D_FILES})
list(APPEND DUMPTEST_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/escape-test.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/include-tests.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/use-tests.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/localfiles-test.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/localfiles_dir/localfiles-compatibility-test.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allexpressions.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allfunctions.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allmodules.scad)

list(APPEND CGALPNGTEST_2D_FILES ${FEATURES_2D_FILES} ${SCAD_DXF_FILES} ${EXAMPLE_2D_FILES})
list(APPEND CGALPNGTEST_3D_FILES ${FEATURES_3D_FILES} ${DEPRECATED_3D_FILES} ${ISSUES_3D_FILES}
 ${EXAMPLE_3D_FILES})
list(APPEND CGALPNGTEST_3D_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/include-tests.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/use-tests.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/transform-nan-inf-tests.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/localfiles-test.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/localfiles_dir/localfiles-compatibility-test.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/rotate-empty-bbox.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/empty-shape-tests.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/null-polygons.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/internal-cavity.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/internal-cavity-polyhedron.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/bad-stl-pcbvicebar.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/bad-stl-tardis.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/bad-stl-wing.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/rotate_extrude-hole.scad
                           ${CMAKE_SOURCE_DIR}/../testdata/scad/experimental/tessellation-text-test.scad)

list(APPEND CGALPNGTEST_FILES ${CGALPNGTEST_2D_FILES} ${CGALPNGTEST_3D_FILES})
list(APPEND OPENCSGTEST_FILES ${CGALPNGTEST_FILES})
list(APPEND OPENCSGTEST_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/intersection-prune-test.scad)
list(APPEND THROWNTOGETHERTEST_FILES ${OPENCSGTEST_FILES})

list(APPEND CGALSTLSANITYTEST_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/normal-nan.scad)

file(GLOB EXPERIMENTAL_TEXT_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/experimental/text-*.scad)
# Issue #899
set_test_config(Bugs dxfpngtest_text-font-direction-tests
                     cgalpngtest_text-font-direction-tests
                     opencsgtest_text-font-direction-tests
                     csgpngtest_text-font-direction-tests
                     throwntogethertest_text-font-direction-tests)

list(APPEND EXPORT3D_TEST_FILES ${EXAMPLE_3D_FILES})
list(APPEND EXPORT3D_TEST_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/polyhedron-nonplanar-tests.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/polyhedron-tests.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/rotate_extrude-tests.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/union-coincident-test.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/null-polygons.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/internal-cavity.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/internal-cavity-polyhedron.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/bad-stl-pcbvicebar.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/bad-stl-tardis.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/bad-stl-wing.scad
                                ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/rotate_extrude-hole.scad)

disable_tests(
  # This doesn't output anything
  dxfpngtest_nothing-decimal-comma-separated

  # Not useful
  throwntogethertest_internal-cavity
  throwntogethertest_internal-cavity-polyhedron

  # these take too long, for little relative gain in testing
  stlpngtest_iteration
  offpngtest_iteration
  stlpngtest_fractal
  offpngtest_fractal
  stlpngtest_logo_and_text
  offpngtest_logo_and_text
)

# 2D tests
list(APPEND FILES_2D ${FEATURES_2D_FILES} ${SCAD_DXF_FILES} ${EXAMPLE_2D_FILES} ${EXPERIMENTAL_TEXT_FILES})
list(APPEND ALL_2D_FILES ${FILES_2D})

# FIXME: This test illustrates a weakness in child() combined with modifiers.
# Reenable it when this is improved
disable_tests(opencsgtest_child-background)

# These tests only makes sense in OpenCSG mode
disable_tests(cgalpngtest_child-background
              cgalpngtest_highlight-and-background-modifier
              cgalpngtest_testcolornames
              csgpngtest_child-background
              csgpngtest_highlight-and-background-modifier
              csgpngtest_testcolornames
              throwntogethertest_testcolornames)

# This test won't render anything meaningful in throwntogether mode
disable_tests(throwntogethertest_minkowski3-erosion)

# The inf/nan tests fail when exporting CSG and rendering that output again
# as currently inf/nan is written directly to the CSG file (e.g. r = inf)
# which is not valid or even misleading in case a variable inf exists.
# FIXME: define export behavior for inf/nan when exporting CSG files
disable_tests(csgpngtest_primitive-inf-tests csgpngtest_transform-nan-inf-tests)

# Test config handling

# Heavy tests are tests taking more than 10 seconds on a development computer
set_test_config(Heavy cgalpngtest_rotate_extrude-tests
                      csgpngtest_rotate_extrude-tests
                      cgalpngtest_for-nested-tests
                      csgpngtest_for-nested-tests
                      cgalpngtest_resize-tests
                      cgalpngtest_text-search-test
                      cgalpngtest_fractal
                      csgpngtest_fractal
                      cgalpngtest_iteration
                      csgpngtest_iteration
                      cgalpngtest_linear_extrude-scale-zero-tests
                      csgpngtest_linear_extrude-scale-zero-tests
                      cgalpngtest_sphere-tests
                      csgpngtest_resize-tests
                      csgpngtest_resize-tests
                      stlpngtest_fence
                      stlpngtest_surface
                      stlpngtest_demo_cut
                      stlpngtest_search
                      stlpngtest_rounded_box
                      stlpngtest_difference
                      stlpngtest_translation
                      offpngtest_fence
                      offpngtest_surface
                      offpngtest_demo_cut
                      offpngtest_search
                      offpngtest_rounded_box
                      offpngtest_difference
                      offpngtest_translation
)

# Bugs

list(APPEND BUGS_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue584.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue591.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue666.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue791.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue802.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue835.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue899.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue904.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue911.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue913.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue936.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue945.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue964.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue964b.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue990.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue1004.scad
                       ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue1005.scad)
list(APPEND EXPORT3D_TEST_FILES ${BUGS_FILES})
list(REMOVE_ITEM EXPORT3D_TEST_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue899.scad)
list(APPEND ALL_2D_FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/bugs/issue899.scad)

list(APPEND OPENCSGTEST_FILES ${BUGS_FILES})
list(APPEND CGALPNGTEST_FILES ${BUGS_FILES})
foreach(FILE ${BUGS_FILES})
  get_test_fullname(opencsgtest ${FILE} TEST_FULLNAME)
  set_test_config(Bugs ${TEST_FULLNAME})
  get_test_fullname(cgalpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Bugs ${TEST_FULLNAME})
  get_test_fullname(csgpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Bugs ${TEST_FULLNAME})
  get_test_fullname(offpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Bugs ${TEST_FULLNAME})
  get_test_fullname(monotonepngtest ${FILE} TEST_FULLNAME)
  set_test_config(Bugs ${TEST_FULLNAME})
  get_test_fullname(stlpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Bugs ${TEST_FULLNAME})
endforeach()

# Examples

foreach(FILE ${EXAMPLE_FILES})
  get_test_fullname(cgalpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Examples ${TEST_FULLNAME})
  get_test_fullname(opencsgtest ${FILE} TEST_FULLNAME)
  set_test_config(Examples ${TEST_FULLNAME})
  get_test_fullname(throwntogethertest ${FILE} TEST_FULLNAME)
  set_test_config(Examples ${TEST_FULLNAME})
  get_test_fullname(csgpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Examples ${TEST_FULLNAME})
  get_test_fullname(monotonepngtest ${FILE} TEST_FULLNAME)
  set_test_config(Examples ${TEST_FULLNAME})
  get_test_fullname(stlpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Examples ${TEST_FULLNAME})
  get_test_fullname(offpngtest ${FILE} TEST_FULLNAME)
  set_test_config(Examples ${TEST_FULLNAME})
endforeach()

# Workaround Gallium bugs
if ( ${CMAKE_SYSTEM_PROCESSOR} MATCHES "ppc")
  message(STATUS "Workaround PPC bug https://bugs.freedesktop.org/show_bug.cgi?id=42540")
  set(CTEST_ENVIRONMENT "${CTEST_ENVIRONMENT};GALLIUM_DRIVER=softpipe")
  set(CTEST_ENVIRONMENT "${CTEST_ENVIRONMENT};DRAW_USE_LLVM=no")
endif()

# Set up custom commands to run before & after Ctest run.
# 1. Start/stop Virtual Framebuffer for linux/bsd. 2. Pretty Print
# Please see the CTestCustom.template file for more info. 

#
# Post-test pretty print
#

add_executable(test_pretty_print test_pretty_print.cc)
file(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/test_pretty_print.py PYSRC)
set_target_properties(test_pretty_print PROPERTIES COMPILE_FLAGS
  "-DPYBIN=${PYTHON_EXECUTABLE} -DPYSRC=${PYSRC} -DBUILDDIR=--builddir=${CMAKE_CURRENT_BINARY_DIR}"
)
#if (MINGW_CROSS_ENV_DIR)
#  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/test_pretty_print "test_pretty_print.exe")
#  execute_process(COMMAND chmod ugo+x ${CMAKE_CURRENT_BINARY_DIR}/test_pretty_print)
#endif()

file(READ ${CMAKE_CURRENT_SOURCE_DIR}/CTestCustom.template TMP)
string(REPLACE __cmake_current_binary_dir__ ${CMAKE_CURRENT_BINARY_DIR} TMP ${TMP})
string(REPLACE __cmake_current_source_dir__ ${CMAKE_CURRENT_SOURCE_DIR} TMP ${TMP})
string(REPLACE __python__ ${PYTHON_EXECUTABLE} TMP ${TMP})
string(REPLACE __header__ "Generated by cmake from ${CMAKE_CURRENT_SOURCE_DIR}/CTestCustom.template" TMP ${TMP})
string(REPLACE __cmake_system_name__ ${CMAKE_SYSTEM_NAME} TMP ${TMP})
string(REPLACE __openscad_binpath__ ${OPENSCAD_BINPATH} TMP ${TMP})

set(OPENSCAD_UPLOAD_TESTS $ENV{OPENSCAD_UPLOAD_TESTS})
set(UPLOADARG "")
if (OPENSCAD_UPLOAD_TESTS)
  set(UPLOADARG "--upload")
endif()
string(REPLACE __openscad_upload_tests__ "${UPLOADARG}" TMP ${TMP})

message(STATUS "creating CTestCustom.cmake")
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/CTestCustom.cmake ${TMP})

#
# Add tests
#
# Types of tests:
# o echotest: Just record console output
# o dumptest: Export .csg
# o cgalpngtest: Export to PNG using --render
# o opencsgtest: Export to PNG using OpenCSG
# o throwntogethertest: Export to PNG using the Throwntogether renderer
# o csgpngtest: 1) Export to .csg, 2) import .csg and export to PNG (--render)
# o monotonepngtest: Same as cgalpngtest but with the "Monotone" color scheme
# o stlpngtest: Export to STL, Re-import and render to PNG (--render=cgal)
# o offpngtest: Export to OFF, Re-import and render to PNG (--render=cgal)
# o dxfpngtest: Export to DXF, Re-import and render to PNG (--render=cgal)
#

add_cmdline_test(moduledumptest EXE ${OPENSCAD_BINPATH} ARGS -o SUFFIX ast FILES
                 ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allmodules.scad
                 ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allfunctions.scad
                 ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allexpressions.scad)
add_cmdline_test(csgtexttest SUFFIX txt FILES
                             ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allexpressions.scad
                             ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allfunctions.scad
                             ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allmodules.scad)
add_cmdline_test(csgtermtest EXE ${OPENSCAD_BINPATH} ARGS -o SUFFIX term FILES
                             ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allexpressions.scad
                             ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allfunctions.scad
                             ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/allmodules.scad)
add_cmdline_test(echotest EXE ${OPENSCAD_BINPATH} ARGS -o SUFFIX echo FILES ${ECHO_FILES})
add_cmdline_test(dumptest EXE ${OPENSCAD_BINPATH} ARGS -o SUFFIX csg FILES ${DUMPTEST_FILES})
add_cmdline_test(dumptest-examples EXE ${OPENSCAD_BINPATH} ARGS -o SUFFIX csg FILES ${EXAMPLE_FILES})
add_cmdline_test(cgalpngtest EXE ${OPENSCAD_BINPATH} ARGS --enable=text --render -o SUFFIX png FILES ${CGALPNGTEST_FILES})
add_cmdline_test(opencsgtest EXE ${OPENSCAD_BINPATH} ARGS --enable=text -o SUFFIX png FILES ${OPENCSGTEST_FILES})
add_cmdline_test(csgpngtest EXE ${PYTHON_EXECUTABLE} SCRIPT ${CMAKE_SOURCE_DIR}/export_import_pngtest.py ARGS --openscad=${OPENSCAD_BINPATH} --format=csg --render EXPECTEDDIR cgalpngtest SUFFIX png FILES ${CGALPNGTEST_FILES})
add_cmdline_test(throwntogethertest EXE ${OPENSCAD_BINPATH} ARGS --preview=throwntogether -o SUFFIX png FILES ${THROWNTOGETHERTEST_FILES})
# FIXME: We don't actually need to compare the output of cgalstlsanitytest
# with anything. It's self-contained and returns != 0 on error
add_cmdline_test(cgalstlsanitytest EXE ${CMAKE_SOURCE_DIR}/cgalstlsanitytest SUFFIX txt ARGS ${OPENSCAD_BINPATH} FILES ${CGALSTLSANITYTEST_FILES})

#
# Export/Import tests
#

# Issue #337
set_test_config(Bugs stlpngtest_polyhedron-tests)
# Issue #910
set_test_config(Bugs offpngtest_polyhedron-tests
                     offpngtest_bad-stl-pcbvicebar
                     offpngtest_bad-stl-tardis
                     offpngtest_bad-stl-wing)

add_cmdline_test(monotonepngtest EXE ${OPENSCAD_BINPATH} ARGS --colorscheme=Monotone --enable=text --render -o SUFFIX png FILES ${EXPORT3D_TEST_FILES})
add_cmdline_test(stlpngtest EXE ${PYTHON_EXECUTABLE} SCRIPT ${CMAKE_SOURCE_DIR}/export_import_pngtest.py ARGS --openscad=${OPENSCAD_BINPATH} --format=STL --render=cgal EXPECTEDDIR monotonepngtest SUFFIX png FILES ${EXPORT3D_TEST_FILES})
add_cmdline_test(offpngtest EXE ${PYTHON_EXECUTABLE} SCRIPT ${CMAKE_SOURCE_DIR}/export_import_pngtest.py ARGS --openscad=${OPENSCAD_BINPATH} --format=OFF --render=cgal EXPECTEDDIR monotonepngtest SUFFIX png FILES ${EXPORT3D_TEST_FILES})
add_cmdline_test(dxfpngtest EXE ${PYTHON_EXECUTABLE} SCRIPT ${CMAKE_SOURCE_DIR}/export_import_pngtest.py ARGS --openscad=${OPENSCAD_BINPATH} --format=DXF --render=cgal EXPECTEDDIR cgalpngtest SUFFIX png FILES ${FILES_2D})


#
# Failing tests
#
add_failing_test(stlfailedtest EXE ${PYTHON_EXECUTABLE} SCRIPT ${CMAKE_SOURCE_DIR}/shouldfail.py ARGS --openscad=${OPENSCAD_BINPATH} --retval=1 -o SUFFIX stl FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/empty-union.scad)
add_failing_test(offfailedtest EXE ${PYTHON_EXECUTABLE} SCRIPT ${CMAKE_SOURCE_DIR}/shouldfail.py ARGS --openscad=${OPENSCAD_BINPATH} --retval=1 -o SUFFIX off FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/empty-union.scad)

#
# Add experimental tests
#
add_cmdline_test(dumptest EXE ${OPENSCAD_BINPATH} ARGS --enable=text -o SUFFIX csg FILES ${EXPERIMENTAL_TEXT_FILES})
add_cmdline_test(cgalpngtest EXE ${OPENSCAD_BINPATH} ARGS --render --enable=text -o SUFFIX png FILES ${EXPERIMENTAL_TEXT_FILES})
add_cmdline_test(opencsgtest EXE ${OPENSCAD_BINPATH} ARGS --enable=text -o SUFFIX png FILES ${EXPERIMENTAL_TEXT_FILES})
add_cmdline_test(csgpngtest EXE ${PYTHON_EXECUTABLE} SCRIPT ${CMAKE_SOURCE_DIR}/export_import_pngtest.py ARGS --openscad=${OPENSCAD_BINPATH} --format=csg --render EXPECTEDDIR cgalpngtest SUFFIX png FILES ${EXPERIMENTAL_TEXT_FILES})
add_cmdline_test(throwntogethertest EXE ${OPENSCAD_BINPATH} ARGS --preview=throwntogether --enable=text -o SUFFIX png FILES ${EXPERIMENTAL_TEXT_FILES})

# Tests using the actual OpenSCAD binary

# non-ASCII filenames
add_cmdline_test(openscad-nonascii EXE ${OPENSCAD_BINPATH} ARGS -o 
                 SUFFIX csg 
                 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/sfære.scad)


# Variable override (-D arg)

# FIXME - this breaks on older cmake that is very common 'in the wild' on linux
# Override simple variable
if("${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}" VERSION_GREATER 2.8.10)
add_cmdline_test(openscad-override EXE ${OPENSCAD_BINPATH}
                 ARGS -D a=3$<SEMICOLON> -o
                 SUFFIX echo
                 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/misc/override.scad)
endif()

# Image output parameters
add_cmdline_test(openscad-imgsize EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize 100,100 -o 
                 SUFFIX png 
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
add_cmdline_test(openscad-imgstretch EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize 500,100 -o 
                 SUFFIX png 
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
add_cmdline_test(openscad-imgstretch2 EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize 100,500 -o 
                 SUFFIX png 
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective gimbal cam
add_cmdline_test(openscad-camdist EXE ${OPENSCAD_BINPATH} 
                 ARGS --imgsize=500,500 --camera=0,0,0,90,0,90,100 -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective gimbal cam
add_cmdline_test(openscad-camrot EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=0,0,0,440,337.5,315,100 -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective gimbal cam
add_cmdline_test(openscad-camtrans EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=100,-20,-10,90,0,90,100 -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective gimbal cam, viewall
add_cmdline_test(openscad-camtrans-viewall EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=100,-20,-10,90,0,90,3000 --viewall -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Orthographic gimbal cam
add_cmdline_test(openscad-camortho EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=100,-20,-20,90,0,90,90 --projection=o -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Orthographic gimbal cam viewall
add_cmdline_test(openscad-camortho-viewall EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=100,-20,-10,90,0,90,3000 --viewall --projection=o -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective vector cam
add_cmdline_test(openscad-cameye EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=60,40,30,0,0,0 -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective vector cam
add_cmdline_test(openscad-cameye2 EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=160,140,130,0,0,0 -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective vector cam
add_cmdline_test(openscad-camcenter EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=60,40,30,20,10,30  -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Perspective vector cam viewall
add_cmdline_test(openscad-camcenter-viewall EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=60,40,30,20,10,30 --viewall -o
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Orthographic vector cam
add_cmdline_test(openscad-cameyeortho EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=160,140,130,0,0,0 --projection=o -o 
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)
# Orthographic vector cam viewall
add_cmdline_test(openscad-cameyeortho-viewall EXE ${OPENSCAD_BINPATH}
                 ARGS --imgsize=500,500 --camera=16,14,13,0,0,0 --viewall --projection=o -o 
                 SUFFIX png
		 FILES ${CMAKE_SOURCE_DIR}/../testdata/scad/3D/features/camera-tests.scad)

# Colorscheme tests
add_cmdline_test(openscad-colorscheme-cornfield EXE ${OPENSCAD_BINPATH}
                 ARGS --colorscheme=Cornfield -o 
                 SUFFIX png 
                 FILES ${CMAKE_SOURCE_DIR}/../examples/Basics/difference_cube.scad)
add_cmdline_test(openscad-colorscheme-metallic EXE ${OPENSCAD_BINPATH}
                 ARGS --colorscheme=Metallic -o 
                 SUFFIX png 
                 FILES ${CMAKE_SOURCE_DIR}/../examples/Basics/difference_cube.scad)
add_cmdline_test(openscad-colorscheme-metallic-render EXE ${OPENSCAD_BINPATH}
                 ARGS --colorscheme=Metallic --render -o 
                 SUFFIX png 
                 FILES ${CMAKE_SOURCE_DIR}/../examples/Basics/difference_cube.scad)
add_cmdline_test(openscad-colorscheme-sunset EXE ${OPENSCAD_BINPATH}
                 ARGS --colorscheme=Sunset -o 
                 SUFFIX png 
                 FILES ${CMAKE_SOURCE_DIR}/../examples/Basics/difference_cube.scad)
add_cmdline_test(openscad-colorscheme-starnight EXE ${OPENSCAD_BINPATH}
                 ARGS --colorscheme=Starnight -o 
                 SUFFIX png 
                 FILES ${CMAKE_SOURCE_DIR}/../examples/Basics/difference_cube.scad)
add_cmdline_test(openscad-colorscheme-monotone EXE ${OPENSCAD_BINPATH}
                 ARGS --colorscheme=Monotone -o 
                 SUFFIX png 
                 FILES ${CMAKE_SOURCE_DIR}/../examples/Basics/difference_cube.scad)

#message("Available test configurations: ${TEST_CONFIGS}")
#foreach(CONF ${TEST_CONFIGS})
#  message("${CONF}: ${${CONF}_TEST_CONFIG}")
#endforeach()

message(STATUS "CPPFLAGS: ${CMAKE_CXX_FLAGS}")
