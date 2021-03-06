project(alacarte)

cmake_minimum_required( VERSION 2.8 )
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

#
# Depends
#
find_package(Boost 1.47.0 COMPONENTS system filesystem thread unit_test_framework program_options serialization regex iostreams REQUIRED)

if (NOT Boost_FOUND)
    message(FATAL_ERROR "Fatal error: Boost (version >= 1.47.0) required.\n")
endif (NOT Boost_FOUND)

#find other packages
IF(WIN32)
	SET(ENV{FREETYPE_DIR} ${CMAKE_FIND_ROOT_PATH})
ELSE(WIN32)
    find_package(Inotify REQUIRED)
ENDIF(WIN32)

find_package(Cairo REQUIRED)
find_package(Freetype REQUIRED)
find_package(Log4Cpp REQUIRED)

find_package(Threads REQUIRED)
find_package(Doxygen)

# FIXME hack for log4cpp backward compability
if (NOT Log4Cpp_LIBRARIES)
    set (Log4Cpp_LIBRARIES "${LOG4CPP_LIBRARIES}")
    set (Log4Cpp_DEFINITIONS "${LOG4CPP_DEFINITIONS}")
    set (Log4Cpp_INCLUDE_DIRS "${LOG4CPP_INCLUDE_DIRS}")
endif()

#
# Build options
#

# FIXME hack for old cairo and MinGW
if(CAIRO_VERSION_V LESS 1 OR CAIRO_VERSION_MAJOR LESS 12 OR CAIRO_VERSION_MINOR LESS 2 OR MINGW)
    OPTION(RENDER_LOCK "Render lock for old cairo and Windows" ON)
    if(MINGW)
        message (STATUS "MinGW detected: enabling render lock.")
    else(MINGW)
        message (STATUS "${Cairo_VERSION} < 1.12.2: enabling render lock.")
    endif(MINGW)
else(CAIRO_VERSION_V LESS 1 OR CAIRO_VERSION_MAJOR LESS 12 OR CAIRO_VERSION_MINOR LESS 2 OR MINGW)
    OPTION(RENDER_LOCK "Render lock for old cairo and Windows" OFF)
endif(CAIRO_VERSION_V LESS 1 OR CAIRO_VERSION_MAJOR LESS 12 OR CAIRO_VERSION_MINOR LESS 2 OR MINGW)
if(RENDER_LOCK)
    set(RENDER_LOCK 1)
else(RENDER_LOCK)
    set(RENDER_LOCK 0)
endif (RENDER_LOCK)

# Set windows version for MinGW and Inotify for non-mingw targets
IF(MINGW)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_WIN32_WINNT=0x0501")
    add_definitions(-D_WIN32_WINNT=0x0501)
    set(SYSTEM_LIBRARIES ws2_32 wsock32)
ELSE(MINGW)
    include_directories(${INOTIFY_INCLUDE_DIRS})
    set(SYSTEM_LIBRARIES ${INOTIFY_LIBRARY})
ENDIF(MINGW)

set(TEST_DIRECTORY "${CMAKE_SOURCE_DIR}/tests")

link_directories(${Boost_LIBRARY_DIRS})
include_directories(include ${Boost_INCLUDE_DIRS} ${CAIRO_INCLUDE_DIR} ${FREETYPE_INCLUDE_DIRS} ${Log4Cpp_INCLUDE_DIRS})

ADD_DEFINITIONS(${Log4Cpp_DEFINITIONS})
ADD_DEFINITIONS(-DStatistic_Activated)

# check for local static build gmock directory
SET(GMOCK_FOUND 0)
IF(EXISTS "${PROJECT_SOURCE_DIR}/gmock" AND IS_DIRECTORY "${PROJECT_SOURCE_DIR}/gmock")
    link_directories(${Boost_LIBRARY_DIRS} ${Cairo_LIBRARY_DIRS} "${PROJECT_SOURCE_DIR}/gmock")
    message("gmock dir found\n")
    SET(GMOCK_FOUND 1)
ENDIF(EXISTS "${PROJECT_SOURCE_DIR}/gmock" AND IS_DIRECTORY "${PROJECT_SOURCE_DIR}/gmock")

if(MSVC)
   ADD_DEFINITIONS(-D_WIN32_WINNT=0x0501)
   ADD_DEFINITIONS(-DNOMINMAX)
   ADD_DEFINITIONS(-D_USE_MATH_DEFINES)
else()
   SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
   SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
   SET(CMAKE_CXX_FLAGS_PROFILE "${CMAKE_CXX_FLAGS_DEBUG} -O0 -ftest-coverage -fprofile-arcs -fprofile-values -fvpt -pg")
   SET(CMAKE_CXX_FLAGS_SANITIZE "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=thread -fno-omit-frame-pointer")
   ADD_DEFINITIONS("-std=c++0x")
   #ADD_DEFINITIONS("-fdiagnostics-color=auto")
   ADD_DEFINITIONS(-DBOOST_TEST_DYN_LINK)
endif()

MARK_AS_ADVANCED(CMAKE_CXX_FLAGS_PROFILE)
SET( CMAKE_BUILD_TYPE "${CMAKE_BUILD_TYPE}" CACHE STRING
    "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel Profile Sanitize."
    FORCE )

set(DEBUG_BUILD 0)
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(DEBUG_BUILD 1)
endif ()
if (CMAKE_BUILD_TYPE STREQUAL "Profile")
    set(DEBUG_BUILD 1)
endif ()
if (CMAKE_BUILD_TYPE STREQUAL "Sanitize")
    set(DEBUG_BUILD 1)
endif ()
message (STATUS "Enabling debug code: ${DEBUG_BUILD}")

message(STATUS "Default: ${CMAKE_CXX_FLAGS}")
message(STATUS "Debug: ${CMAKE_CXX_FLAGS_DEBUG}")
message(STATUS "Profile: ${CMAKE_CXX_FLAGS_PROFILE}")
message(STATUS "Release: ${CMAKE_CXX_FLAGS_RELEASE}")
message(STATUS "Sanitize: ${CMAKE_CXX_FLAGS_SANITIZE}")

#
# Sources
#
file(GLOB_RECURSE server_sources                src/server/*.cpp)
file(GLOB_RECURSE importer_sources              src/importer/*.cpp)
file(GLOB_RECURSE alacarte_sources              src/general/*.cpp src/utils/*.cpp)
file(GLOB_RECURSE UnitTests_general_sources     tests/general/*.cpp)
file(GLOB_RECURSE UnitTests_server_sources      tests/server/*.cpp)
file(GLOB_RECURSE UnitTests_mapcss_sources      tests/mapcss/*.cpp)
file(GLOB_RECURSE UnitTests_importer_sources    tests/importer/*.cpp)
file(GLOB_RECURSE UnitTests_utils_sources       tests/utils/*.cpp)
file(GLOB_RECURSE UnitTests_eval_sources        tests/eval/*.cpp)
file(GLOB_RECURSE UnitTests_parser_sources      tests/parser/*.cpp)
file(GLOB_RECURSE UnitTests_shared_sources      tests/shared/*.cpp)

#
# Targets
#
configure_file("data/config/alacarte.conf.in" "${CMAKE_SOURCE_DIR}/data/config/alacarte.conf")
configure_file("tests/config.hpp.in" "${CMAKE_SOURCE_DIR}/tests/config.hpp")
configure_file("include/config.hpp.in" "${CMAKE_SOURCE_DIR}/include/config.hpp")

add_library(alacarte-obj OBJECT ${alacarte_sources})
add_library(server-obj OBJECT ${server_sources})

add_executable(alacarte-server      src/alacarte_server.cpp     $<TARGET_OBJECTS:server-obj> $<TARGET_OBJECTS:alacarte-obj>)
add_executable(alacarte-importer    src/alacarte_importer.cpp   ${importer_sources} $<TARGET_OBJECTS:alacarte-obj>)

add_executable(unitTests_utils      EXCLUDE_FROM_ALL ${UnitTests_utils_sources} $<TARGET_OBJECTS:alacarte-obj> ${UnitTests_shared_sources})
add_executable(unitTests_parser     EXCLUDE_FROM_ALL ${UnitTests_parser_sources} $<TARGET_OBJECTS:alacarte-obj> $<TARGET_OBJECTS:server-obj> ${UnitTests_shared_sources})
add_executable(unitTests_general    EXCLUDE_FROM_ALL ${UnitTests_general_sources} ${alacarte_sources} ${UnitTests_shared_sources})
add_executable(unitTests_importer   EXCLUDE_FROM_ALL ${UnitTests_importer_sources} ${UnitTests_shared_sources} $<TARGET_OBJECTS:alacarte-obj>)
add_executable(unitTests_server     EXCLUDE_FROM_ALL ${UnitTests_server_sources} ${UnitTests_shared_sources} ${server_sources} $<TARGET_OBJECTS:alacarte-obj>)
IF(GMOCK_FOUND)
    add_executable(unitTests_mapcss EXCLUDE_FROM_ALL ${UnitTests_mapcss_sources} ${UnitTests_shared_sources} $<TARGET_OBJECTS:server-obj> $<TARGET_OBJECTS:alacarte-obj>)
ENDIF(GMOCK_FOUND)
add_executable(unitTests_eval       EXCLUDE_FROM_ALL ${UnitTests_eval_sources} ${UnitTests_shared_sources} $<TARGET_OBJECTS:server-obj> $<TARGET_OBJECTS:alacarte-obj>)


IF(BUILD_TESTS)
   enable_testing()
   add_test(unitTests_utils unitTests_utils)
   add_test(unitTests_eval unitTests_eval)
   add_test(unitTests_parser unitTests_parser)
   add_test(unitTests_general unitTests_general)
   add_test(unitTests_importer unitTests_importer)
   add_test(unitTests_server unitTests_server)
IF(GMOCK_FOUND)
   add_test(unitTests_mapcss unitTests_mapcss)
ENDIF(GMOCK_FOUND)
ENDIF()

set_target_properties(unitTests_eval       PROPERTIES COMPILE_DEFINITIONS "ALACARTE_TEST")
set_target_properties(unitTests_general    PROPERTIES COMPILE_DEFINITIONS "ALACARTE_TEST")
set_target_properties(unitTests_utils      PROPERTIES COMPILE_DEFINITIONS "ALACARTE_TEST")
set_target_properties(unitTests_parser     PROPERTIES COMPILE_DEFINITIONS "ALACARTE_TEST")
set_target_properties(unitTests_eval       PROPERTIES COMPILE_DEFINITIONS "ALACARTE_TEST")
set_target_properties(unitTests_server     PROPERTIES COMPILE_DEFINITIONS "ALACARTE_TEST")
IF(GMOCK_FOUND)
    set_target_properties(unitTests_mapcss PROPERTIES COMPILE_DEFINITIONS "ALACARTE_TEST")
ENDIF(GMOCK_FOUND)

target_link_libraries(alacarte-server       ${Boost_LIBRARIES} ${CAIRO_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${SYSTEM_LIBRARIES})
target_link_libraries(alacarte-importer     ${Boost_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${SYSTEM_LIBRARIES})
target_link_libraries(unitTests_utils       ${Boost_LIBRARIES} ${CAIRO_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(unitTests_general     ${Boost_LIBRARIES} ${CAIRO_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(unitTests_importer    ${Boost_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CAIRO_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(unitTests_server      ${Boost_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CAIRO_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(unitTests_parser      ${Boost_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CAIRO_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
IF(GMOCK_FOUND)
    target_link_libraries(unitTests_mapcss  ${Boost_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CAIRO_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} gmock gtest)
ENDIF(GMOCK_FOUND)
target_link_libraries(unitTests_eval        ${Boost_LIBRARIES} ${Log4Cpp_LIBRARIES} ${CAIRO_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})

set(test_files "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/karlsruhe_big.carte" "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/renderer_test.carte" "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/test_tiles.carte")

add_custom_command(
    OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/karlsruhe_big.carte"
    COMMAND alacarte-importer "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/karlsruhe_big.osm" "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/karlsruhe_big.carte"
)
add_custom_command(
    OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/test_tiles.carte"
    COMMAND alacarte-importer "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/test_tiles.osm" "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/test_tiles.carte"
)
add_custom_command(
    OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/renderer_test.carte"
    COMMAND alacarte-importer "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/renderer_test.osm" "${CMAKE_CURRENT_SOURCE_DIR}/tests/data/input/renderer_test.carte"
)

add_custom_target(make-carto-files
    DEPENDS alacarte-importer ${test_files}
)

add_custom_target(tests
    DEPENDS make-carto-files unitTests_server unitTests_importer unitTests_general unitTests_parser unitTests_utils
)

add_custom_target(lint
	COMMAND cpplint --filter=-whitespace --extensions=c,h,cpp,hpp ${server_sources} ${importer_source} ${alacarte_sources}
	COMMENT "Check if there are style violations" VERBATIM
)

# add a target to generate API documentation with Doxygen
if(DOXYGEN_FOUND)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/doc.doxy ${CMAKE_CURRENT_BINARY_DIR}/doc.doxy @ONLY)
    add_custom_target(doc ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/doc.doxy
        COMMAND convert -density 100 doc/implementierung/gantt.pdf doc/doxygen/gantt.png
        COMMAND xdg-open doc/doxygen/html/index.html
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Generating API documentation with Doxygen" VERBATIM
    )
endif(DOXYGEN_FOUND)

#
# Installation
#
install(TARGETS alacarte-server RUNTIME DESTINATION bin)
install(TARGETS alacarte-importer RUNTIME DESTINATION bin)
install(FILES data/config/alacarte.conf DESTINATION /etc)
install(FILES data/default.png DESTINATION share/alacarte)
install(DIRECTORY data/mapcss/ DESTINATION share/alacarte/styles)
install(DIRECTORY data/img DESTINATION share/alacarte)
install(DIRECTORY data/icons DESTINATION share/alacarte)
