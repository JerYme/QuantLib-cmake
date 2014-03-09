# CMake build system for QuantLib 
# 2012, Joao Paulo Magalhaes <jpmagalhaes@ist.utl.pt>

cmake_minimum_required ( VERSION 2.8 )

project ( QuantLib )
set_property ( GLOBAL PROPERTY USE_FOLDERS ON )

set ( QuantLib_MAJOR_VERSION 1 CACHE STRING "QuantLib major version number." FORCE )
set ( QuantLib_MINOR_VERSION 2 CACHE STRING "QuantLib minor version number." FORCE )
set ( QuantLib_VERSION_STRING "${QuantLib_MAJOR_VERSION}.${QuantLib_MINOR_VERSION}" )

#--------------------------------------------------
# initial settings - cached

OPTION ( QuantLib_BUILD_STATIC "Build a static library." ON )
OPTION ( QuantLib_BUILD_SHARED "Build a shared library." OFF )


OPTION ( QuantLib_ENABLE_INDEXED_COUPONS
"Use indexed coupons in a floating rate leg.
Use par coupons otherwise." OFF )
OPTION ( QuantLib_ENABLE_NEGATIVE_RATES
"Allow negative yield rates in a few places where they are
currently forbidden. It is still not clear whether this is safe." OFF )
OPTION ( QuantLib_ENABLE_EXTRA_SAFETY_CHECKS
"Add extra runtime checks to a few functions. This can
prevent their inlining and degrade performance." OFF )
OPTION ( QuantLib_ENABLE_DEPRECATED_CODE
"Include deprecated code in the library." ON )
OPTION ( QuantLib_ENABLE_SESSIONS
"If enabled, singletons will return different instances for different sessions.
You will have to provide and link with the library a sessionId() function in
namespace QuantLib, returning a different session id for each session." OFF )


OPTION ( QuantLib_ENABLE_MSG_ERROR_LINES
         "Whether to print file and line info in error messages." OFF )
OPTION ( QuantLib_ENABLE_MSG_FUNCTIONS
         "Whether to print function names in error messages." OFF )
OPTION ( QuantLib_ENABLE_MSG_TRACING
         "Whether to print traces in error messages. Degrades performance." OFF )

OPTION ( QuantLib_WITH_EXAMPLES "Whether to build and install examples." OFF )

if ( QuantLib_WITH_EXAMPLES )
  OPTION ( QuantLib_WITH_EXAMPLES_IN_SHARE
"Whether the examples binaries and respective sources shall be put
in the share folder. When OFF, the binaries go to bin and the
source is not copied." OFF )
  # default to OFF to retain current autotools behaviour
endif ()

OPTION ( QuantLib_WITH_BENCHMARK "Whether to build and install benchmarks." OFF )

# Benchmarks require boost::unit_test_framework which must be linkable
if ( QuantLib_WITH_BENCHMARK )
  OPTION ( QuantLib_Boost_USES_MULTITHREADED "Use the multithreaded Boost libraries" OFF )
  OPTION ( QuantLib_Boost_USES_STATIC_LIBS "Use the static Boost libraries" OFF )
  OPTION ( QuantLib_Boost_USES_STATIC_RUNTIME
  "Searches for Boost libraries linked against a static C++ standard library (ABI tag s)." OFF )
endif ()


#--------------------------------------------------
# initial settings - non-cached
if (NOT QuantLib_ROOT_DIR)
	set ( QuantLib_ROOT_DIR  ${PROJECT_SOURCE_DIR} )
	
	# resolve the path without any relative components
	get_filename_component ( tmp ${QuantLib_ROOT_DIR} REALPATH )
	set ( QuantLib_ROOT_DIR  ${tmp} ) 
endif ()


if (NOT QuantLib_SRC_DIR)
	set ( QuantLib_SRC_DIR   ${QuantLib_ROOT_DIR}/ql )
endif ()

set ( QuantLib_EXAMPLES_DIR   ${QuantLib_ROOT_DIR}/Examples )
set ( QuantLib_BENCHMARK_DIR   ${QuantLib_ROOT_DIR}/test-suite )
set ( QuantLib_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR} )

#--------------------------------------------------
# third-party libraries

set ( QuantLib_LIBRARIES )

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
        message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

#--------------------------------
# find BOOST - see http://www.cmake.org/cmake/help/cmake-2-8-docs.html#module:FindBoost
## Boost ####################################################################

set(Boost_USE_MULTITHREADED ON)

if(WIN32)
    set(Boost_USE_STATIC_LIBS ON)
endif()

set(Boost_FIND_REQUIRED TRUE)

OPTION ( Boost_DEBUG "Whether to print debug info when cmake is looking for Boost" OFF )
set ( Boost_DETAILED_FAILURE_MSG TRUE )

set ( Boost_USE_MULTITHREADED ${QuantLib_Boost_USES_MULTITHREADED} )
set ( Boost_USE_STATIC_LIBS ${QuantLib_Boost_USES_STATIC_LIBS} )
set ( Boost_USE_STATIC_RUNTIME ${QuantLib_Boost_USES_STATIC_RUNTIME} )

# C:/Program Files (x86)/CMake 2.8/share/cmake-2.8/Modules/FindBoost.cmake
# find_package ( Boost COMPONENTS ${qlbc} REQUIRED )
find_package ( Boost COMPONENTS REQUIRED )

if(Boost_INCLUDE_DIR)
    include_directories(${Boost_INCLUDE_DIR})
else()
    message(FATAL_ERROR "Boost headers not found. You will have to specify them in your build environment or you can use -DBOOST_ROOT or -DBOOST_INCLUDE_DIR to tell cmake where your boost installation is.")
endif()

if(NOT Boost_FOUND)
    message(FATAL_ERROR "Boost not found!")
endif()

list ( APPEND QuantLib_INCLUDE_DIRS ${Boost_INCLUDE_DIR} )
list ( APPEND QuantLib_LIBRARIES ${Boost_LIBRARIES} )

#---------------------------------------------------------
# deal with configuration & system introspection

include ( config.cmake )

#---------------------------------------------------------
# deal with the source files

# to generate files.cmake, use the following cmd from the QuantLib top dir:
# ( d=`pwd`/ql ; echo "set ( QuantLib_HEADERS " ; find $d -name "*.hpp" | sed "s:$d:  \${QuantLib_ROOT_DIR}/ql:g" | sort ; echo ")" ; echo ; echo ; echo "set(QuantLib_SOURCES " ; find $d -name "*.cpp" -o -name "*.c" | sed "s:$d:  \${QuantLib_ROOT_DIR}/ql:g" | sort ; echo ")" ; echo ) > files.cmake

include ( files.cmake )

# define a macro to create hierarchical source groups based on a dir tree
macro ( create_source_group GroupPrefix RootDir ProjectSources  )
  set ( DirSources ${ProjectSources} )
  foreach ( Source ${DirSources} )
	string ( REPLACE "\\" "/" RootDirEscaped "${RootDir}" )
    string ( REGEX REPLACE "${RootDirEscaped}" "" RelativePath "${Source}" )
    string ( REGEX REPLACE "[\\\\/][^\\\\/]*$" "" RelativePath "${RelativePath}" )
    string ( REGEX REPLACE "^[\\\\/]" "" RelativePath "${RelativePath}" )
    string ( REGEX REPLACE "/" "\\\\\\\\" RelativePath "${RelativePath}" )
    source_group ( "${GroupPrefix}\\${RelativePath}" FILES ${Source} )
  endforeach ( Source )
endmacro ( create_source_group )

create_source_group ( "Header Files" ${QuantLib_SRC_DIR} "${QuantLib_HEADERS}" )
create_source_group ( "Source Files" ${QuantLib_SRC_DIR} "${QuantLib_SOURCES}" )

#---------------------------------------------------------
# create the library

include_directories ( 
  ${QuantLib_ROOT_DIR}
  ${QuantLib_INCLUDE_DIRS}
)

if ( NOT QuantLib_BUILD_STATIC AND NOT QuantLib_BUILD_SHARED ) 
  message ( ERROR "Must build at least one static or shared version" )
endif ()

# convenience target
add_custom_target ( QuantLib )
set_target_properties ( QuantLib PROPERTIES FOLDER "QuantLib" )

# static version -------------------------------------------
if ( QuantLib_BUILD_STATIC )
  add_library ( QuantLib-static STATIC ${QuantLib_SOURCES} ${QuantLib_HEADERS} )
  set_target_properties ( QuantLib-static PROPERTIES
    VERSION ${QuantLib_VERSION_STRING}
    OUTPUT_NAME "QuantLib"
	FOLDER "QuantLib"
  )
  target_link_libraries ( QuantLib-static ${QuantLib_LIBRARIES} )
  add_dependencies ( QuantLib QuantLib-static )
  install ( TARGETS QuantLib-static
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
  )
endif ()

# shared version -------------------------------------------
if ( QuantLib_BUILD_SHARED )
  add_library ( QuantLib-shared SHARED ${QuantLib_SOURCES} ${QuantLib_HEADERS} )
  set_target_properties ( QuantLib-shared PROPERTIES
    VERSION ${QuantLib_VERSION_STRING}
    OUTPUT_NAME "QuantLib"
	FOLDER "QuantLib"
  )
  target_link_libraries ( QuantLib-shared ${QuantLib_LIBRARIES} )
  add_dependencies ( QuantLib QuantLib-shared )
  install ( TARGETS QuantLib-shared
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
  )
endif ()

# install headers ----------------------------------------
install ( DIRECTORY ${QuantLib_SRC_DIR} 
  DESTINATION include 
  FILES_MATCHING PATTERN "*.hpp" PATTERN "*.h"
)

# don't forget this
install ( FILES cmake/FindQuantLib.cmake DESTINATION share/cmake )

#----------------------------------------------------------------------
# examples

if ( QuantLib_WITH_EXAMPLES )
  add_subdirectory ( ${QuantLib_EXAMPLES_DIR} examples )
endif ()

#----------------------------------------------------------------------
# benchmark

if ( QuantLib_WITH_BENCHMARK )
  add_subdirectory ( ${QuantLib_BENCHMARK_DIR} benchmark )
endif ()

