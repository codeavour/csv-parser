cmake_minimum_required(VERSION 3.9)
project(csv)

if(CSV_CXX_STANDARD)
	set(CMAKE_CXX_STANDARD ${CSV_CXX_STANDARD})
else()
	set(CMAKE_CXX_STANDARD 17)
endif(CSV_CXX_STANDARD)

option(BUILD_PYTHON "Build Python Binding" OFF)

message("Building CSV library using C++${CMAKE_CXX_STANDARD}")

# Defines CSV_HAS_CXX17 in compatibility.hpp
if (CMAKE_VERSION VERSION_LESS "3.12.0")
	add_definitions(-DCMAKE_CXX_STANDARD=${CMAKE_CXX_STANDARD})
else()
	add_compile_definitions(CMAKE_CXX_STANDARD=${CMAKE_CXX_STANDARD})
endif()

set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads QUIET REQUIRED)

if(MSVC)
	# Make Visual Studio report accurate C++ version
	# See: https://devblogs.microsoft.com/cppblog/msvc-now-correctly-reports-__cplusplus/
  # /Wall emits warnings about the C++ standard library
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc /GS- /Zc:__cplusplus /W4")
else()
	# Ignore Visual Studio pragma regions
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unknown-pragmas")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} --coverage -Og")
endif(MSVC)

set(CSV_ROOT_DIR ${CMAKE_CURRENT_LIST_DIR})
set(CSV_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(CSV_INCLUDE_DIR ${CMAKE_CURRENT_LIST_DIR}/include/)
set(CSV_SOURCE_DIR ${CSV_INCLUDE_DIR}/internal/)
set(CSV_TEST_DIR ${CMAKE_CURRENT_LIST_DIR}/tests)

include_directories(${CSV_INCLUDE_DIR})

## Load developer specific CMake settings
if (CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    SET(CSV_DEVELOPER TRUE)
endif()

## Main Library
add_subdirectory(${CSV_SOURCE_DIR})

# build the python binding for the library 
if (${BUILD_PYTHON})
    message("Building Python bindings for the library.")
    add_subdirectory(python)
endif()

## Executables
option(CSV_BUILD_PROGRAMS "Allow to disable building of programs" ON)
if (CSV_BUILD_PROGRAMS)
    add_subdirectory("programs")
endif()

## Developer settings
if (CSV_DEVELOPER)
    # Allow for performance profiling
    if (MSVC)
	    target_link_options(csv PUBLIC /PROFILE)
    endif()
     
    # More error messages.
    if (UNIX)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} \
        -Wall -Wextra -Wsign-compare \
        -Wwrite-strings -Wpointer-arith -Winit-self \
        -Wconversion -Wno-sign-conversion")
    endif()

    # Generate a single header library
    if(CMAKE_VERSION VERSION_LESS "3.12")
      find_package(PythonInterp 3 QUIET)
    else()
      find_package(Python3 COMPONENTS Interpreter)
    endif()
    if(Python3_Interpreter_FOUND OR PYTHONINTERP_FOUND)
      add_custom_target(generate_single_header
          COMMAND ${Python3_EXECUTABLE} single_header.py > single_include/csv.hpp
          COMMAND ${Python3_EXECUTABLE} single_header.py > single_include_test/csv.hpp
          WORKING_DIRECTORY ${CSV_ROOT_DIR}
      )
      # Single header compilation test
      add_subdirectory(single_include_test)
    else()
      message(WARNING "Python3 not found, skipping target 'generate_single_header'.")
    endif()

    # Documentation
    find_package(Doxygen QUIET)
    if(DOXYGEN_FOUND)
      add_custom_target(doxygen
          COMMAND ${DOXYGEN_EXECUTABLE} ${CSV_ROOT_DIR}/Doxyfile
          WORKING_DIRECTORY ${CSV_ROOT_DIR}
      )
    else()
      message(WARNING "Doxygen not found, skipping target 'doxygen'.")
    endif()

    ## Tests
    enable_testing()
    add_subdirectory("tests")

    # Code coverage
    #find_program( GCOV_PATH gcov )
    #if(GCOV_PATH)
    #    set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/modules")
    #    include(CodeCoverage)
    #    append_coverage_compiler_flags()
    #    set(ENV{CSV_TEST_ROOT} ${CSV_TEST_DIR})
    #    setup_target_for_coverage_gcovr_html(
    #      NAME csv_coverage
    #      EXECUTABLE csv_test
    #      EXCLUDE "tests/*"
    #    )
    #endif()
endif()

add_library(csv_parser INTERFACE)
target_include_directories(csv_parser INTERFACE
        $<INSTALL_INTERFACE:include>
)

option(CSV_SINGLE_INCLUDE "Use single include" ON)
if (CSV_SINGLE_INCLUDE)
    set(CSV_INCLUDES_DIR "single_include/")
else ()
    set(CSV_INCLUDES_DIR "include/")
endif()

message("Using single include: ${CSV_SINGLE_INCLUDE}")

install(DIRECTORY ${CSV_INCLUDES_DIR} DESTINATION include/csv_parser)

install(TARGETS csv_parser
        EXPORT csv_parserTargets
)

include(CMakePackageConfigHelpers)

# write_basic_package_version_file(
#         "${CMAKE_CURRENT_SOURCE_DIR}/cmake/csv_parserConfigVersion.cmake"
#         VERSION ${PROJECT_VERSION}
#         COMPATIBILITY SameMajorVersion
# )

configure_package_config_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/csv_parserConfig.cmake.in"
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/csv_parserConfig.cmake"
        INSTALL_DESTINATION lib/cmake/csv_parser
)

install(FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/csv_parserConfig.cmake"
        #        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/csv_parserConfigVersion.cmake"
        DESTINATION lib/cmake/csv_parser
)

install(EXPORT csv_parserTargets
        NAMESPACE csv::
        DESTINATION lib/cmake/csv_parser
)