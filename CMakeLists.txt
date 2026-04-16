# t. schneider tes@mit.edu 
# base CMakeLists.txt file for dccl
# for help on CMake, see http://www.cmake.org/cmake/help/documentation.html
# or type > man cmake

# this project is intended to be built out of source by using 
# > ./build.sh

cmake_minimum_required(VERSION 3.25 FATAL_ERROR)

project(dccl)

find_package(Boost REQUIRED COMPONENTS headers)

message(STATUS "Build host architecture: ${CMAKE_HOST_SYSTEM_PROCESSOR}")
message(STATUS "Build target architecture: ${CMAKE_SYSTEM_PROCESSOR}")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/")

# enable sanitizers (e.g., Address Sanitizer)
find_package(Sanitizers)

# enable testing (CTest)
option(enable_testing "Enable building of tests using CTest (if set to ON, you can run tests with 'make test')" OFF)
if(enable_testing)
  include(CTest)
endif()

option(build_apps "Build supporting tools and other binary applications" ON)

file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/version.txt DCCL_VERSION_STRING)
string(REGEX REPLACE "^([0-9]+).*$" "\\1" DCCL_VERSION_MAJOR "${DCCL_VERSION_STRING}")
string(REGEX REPLACE "^[0-9]+\\.([0-9]+).*$" "\\1" DCCL_VERSION_MINOR  "${DCCL_VERSION_STRING}")
string(REGEX REPLACE "^[0-9]+\\.[0-9]+\\.([0-9~a-z]+).*$" "\\1" DCCL_VERSION_PATCH "${DCCL_VERSION_STRING}")

find_package(Git)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git" AND GIT_FOUND)
   message(STATUS "Compiling in Git source tree.")
   execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
     WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
     OUTPUT_VARIABLE DCCL_LAST_REV)
   string(STRIP ${DCCL_LAST_REV} DCCL_LAST_REV)

   string(REPLACE "~" "_" DCCL_GIT_VERSION ${DCCL_VERSION_MAJOR}.${DCCL_VERSION_MINOR}.${DCCL_VERSION_PATCH}) 

   execute_process(COMMAND ${GIT_EXECUTABLE} rev-list ${DCCL_GIT_VERSION}..HEAD --count
     WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
     OUTPUT_VARIABLE DCCL_REVS_SINCE_TAG)
   string(STRIP ${DCCL_REVS_SINCE_TAG} DCCL_REVS_SINCE_TAG)
      
   execute_process(COMMAND ${GIT_EXECUTABLE} diff-index --quiet HEAD
     WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
     RESULT_VARIABLE DCCL_DIRTY_REV)
   if(DCCL_DIRTY_REV EQUAL 0)
     set(DCCL_DIRTY_REV_STRING "")
   else()
     set(DCCL_DIRTY_REV_STRING "-dirty")
   endif()
   set(DCCL_VERSION_PATCH "${DCCL_VERSION_PATCH}+${DCCL_REVS_SINCE_TAG}+git${DCCL_LAST_REV}${DCCL_DIRTY_REV_STRING}")
   
else()
   message(STATUS "Compiling from release tarball.")
   set(DCCL_VERSION_DATE "2026.03.25")
endif()

set(DCCL_VERSION "${DCCL_VERSION_MAJOR}.${DCCL_VERSION_MINOR}.${DCCL_VERSION_PATCH}")


# give Dccl 2 series a few more soversions
set(DCCL_SOVERSION "31")

# fetch all the local directories for generated code
get_filename_component(dccl_SRC_DIR src ABSOLUTE)
set(dccl_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
get_filename_component(dccl_SCRIPTS_DIR scripts ABSOLUTE)

get_filename_component(dccl_LIB_DIR ${dccl_BUILD_DIR}/lib ABSOLUTE)
get_filename_component(dccl_BIN_DIR ${dccl_BUILD_DIR}/bin ABSOLUTE)
get_filename_component(dccl_INC_DIR ${dccl_BUILD_DIR}/include ABSOLUTE)
get_filename_component(dccl_SHARE_DIR ${dccl_BUILD_DIR}/share ABSOLUTE)

get_filename_component(dccl_PYTHON_DIR python ABSOLUTE)

if(CMAKE_CROSSCOMPILING)
  message(STATUS "DCCL is cross-compiling. Target library architecture: ${CMAKE_LIBRARY_ARCHITECTURE}")
  find_program(dccl_PROTOC_GEN_EXEC protoc-gen-dccl)
  if(dccl_PROTOC_GEN_EXEC)
    get_filename_component(dccl_DEFAULT_EXEC_DIR ${dccl_PROTOC_GEN_EXEC} DIRECTORY)
  else()
    message(FATAL_ERROR "Failed to find protoc-gen-dccl for cross-compiling")
  endif()
else()
  set(dccl_DEFAULT_EXEC_DIR "${dccl_BIN_DIR}")
endif()
  
set(dccl_EXEC_DIR "${dccl_DEFAULT_EXEC_DIR}" CACHE STRING "Directory where executable DCCL code resides. Change for cross-compilation.")

# symlink these into the build directory for backwards compatibility
option(ADD_SYMLINKS "Add symlinks to root directory for backwards compatibility." OFF)
if (ADD_SYMLINKS)
  get_filename_component(dccl_FORMER_LIB_DIR lib ABSOLUTE)
  get_filename_component(dccl_FORMER_BIN_DIR bin ABSOLUTE)
  get_filename_component(dccl_FORMER_INC_DIR include ABSOLUTE)
  get_filename_component(dccl_FORMER_SHARE_DIR share ABSOLUTE)
  add_custom_command(OUTPUT LibLink COMMAND ${CMAKE_COMMAND} -E create_symlink ${dccl_LIB_DIR} ${dccl_FORMER_LIB_DIR})
  add_custom_target(lib_link ALL DEPENDS LibLink)
  add_custom_command(OUTPUT BinLink COMMAND ${CMAKE_COMMAND} -E create_symlink ${dccl_BIN_DIR} ${dccl_FORMER_BIN_DIR})
  add_custom_target(bin_link ALL DEPENDS BinLink)
  add_custom_command(OUTPUT IncludeLink COMMAND ${CMAKE_COMMAND} -E create_symlink ${dccl_INC_DIR} ${dccl_FORMER_INC_DIR})
  add_custom_target(inc_link ALL DEPENDS IncludeLink)
  add_custom_command(OUTPUT ShareLink COMMAND ${CMAKE_COMMAND} -E create_symlink ${dccl_SHARE_DIR} ${dccl_FORMER_SHARE_DIR})
  add_custom_target(share_link ALL DEPENDS ShareLink)
endif()


## set type of libraries
option(make_static_libs "Build static libraries instead of shared." OFF)

if(make_static_libs)
  set(BUILD_SHARED_LIBS OFF)
else()
  set(BUILD_SHARED_LIBS ON)
endif()

## set the cmake defaults for libraries and binaries
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${dccl_LIB_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${dccl_BIN_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${dccl_LIB_DIR})


## set instructions for `make install`

# override library version for debian sbuild cross-builds
if(CMAKE_CROSSCOMPILING)
  if (EXISTS "/etc/debian_version")
    set(CMAKE_INSTALL_LIBDIR "lib/${CMAKE_LIBRARY_ARCHITECTURE}")
  endif()
endif()

include(GNUInstallDirs)

install(DIRECTORY ${dccl_SHARE_DIR}/ DESTINATION ${CMAKE_INSTALL_DATADIR}/dccl)

install(DIRECTORY ${dccl_INC_DIR}/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  FILES_MATCHING PATTERN  "*.h" PATTERN "*.hpp" PATTERN "*.proto" PATTERN "test*" EXCLUDE)

install(EXPORT dccl-targets DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/dccl)

message(STATUS "Installation directories set to: \n\tlibraries: ${CMAKE_INSTALL_FULL_LIBDIR}\n\tbinaries: ${CMAKE_INSTALL_FULL_BINDIR}\n\theaders: ${CMAKE_INSTALL_FULL_INCLUDEDIR}")


set_directory_properties(PROPERTIES ADDITIONAL_CLEAN_FILES
  "${dccl_INC_DIR};${dccl_FORMER_BIN_DIR};${dccl_FORMER_LIB_DIR};${dccl_FORMER_INC_DIR};${dccl_FORMER_SHARE_DIR};${dccl_PYTHON_DIR}/dccl/option_extensions_pb2.py;${dccl_SRC_DIR}/share/examples/quickstart_navreport/navreport_pb2.py;${dccl_SRC_DIR}/share/examples/quickstart_navreport/navreport_pb2.pyc"
  )

# CONFIG mode is required so that protobuf's CMake targets carry transitive
# abseil dependencies.  MODULE_COMPATIBLE enables the legacy variable names
# (PROTOBUF_INCLUDE_DIRS, PROTOBUF_PROTOC_EXECUTABLE, etc.) used by
# DCCLProtobufGenerate.
set(protobuf_MODULE_COMPATIBLE ON CACHE BOOL "" FORCE)
find_package(Protobuf CONFIG REQUIRED)
include(DCCLProtobufGenerate)

message(STATUS "Google Protocol Buffers Version: ${Protobuf_VERSION}")

# optional
## cryptopp
find_package(Cryptopp QUIET)
set(CRYPTOPP_DOC_STRING "Enable cryptography (requires libcrypto++-dev: http://www.cryptopp.com)")
if(CRYPTOPP_FOUND)
  option(enable_cryptography ${CRYPTOPP_DOC_STRING} ON)
else()
  option(enable_cryptography ${CRYPTOPP_DOC_STRING} OFF)
  message(STATUS ">> setting enable_cryptography to OFF ... if you need this functionality: 1) install libcrypto++-dev; 2) run cmake -Denable_cryptography=ON")
endif()

if(enable_cryptography)
  find_package(Cryptopp REQUIRED)
  set(DCCL_HAS_CRYPTOPP "1")
  include_directories(SYSTEM ${Cryptopp_INCLUDE_DIRS})
else()
  set(DCCL_HAS_CRYPTOPP "0")
endif()

## b64 for base64 functions
find_package(B64 QUIET)
set(B64_DOC_STRING "Enable base64 functionality (requires libb64-dev: http://libb64.sourceforge.net/")
if(B64_FOUND)
  option(enable_b64 ${B64_DOC_STRING} ON)
else()
  option(enable_b64 ${B64_DOC_STRING} OFF)
  message(STATUS ">> setting enable_b64 to OFF ... if you need this functionality: 1) install libb64-dev; 2) run cmake -Denable_b64=ON")
endif()

if(enable_b64)
  find_package(B64 REQUIRED)
  set(DCCL_HAS_B64 "1")
  include_directories(SYSTEM ${B64_INCLUDE_DIRS})
else()
  set(DCCL_HAS_B64 "0")
endif()


## LUA for dynamic conditionals
option(enable_lua "Enable LUA functionality (requires liblua5.3-dev)" OFF)

if(enable_lua)
  find_package(Lua REQUIRED)
  set(DCCL_HAS_LUA "1")
  include_directories(SYSTEM ${LUA_INCLUDE_DIR})
else()
  set(DCCL_HAS_LUA "0")
endif()

option(enable_thread_safety "Enable DCCL thread safety (uses std::mutex)" ON)
if(enable_thread_safety)
  set(DCCL_HAS_THREAD_SUPPORT "1")
  # Protobuf uses pthread so we need to link it too (if it's here)
  find_package(Threads)
else()
  set(DCCL_HAS_THREAD_SUPPORT "0")
endif()

## boost for units
set(UNITS_DOC_STRING "Enable static unit-safety functionality (requires Boost)")
if(Boost_FOUND)
  option(enable_units ${UNITS_DOC_STRING} ON) 
else()
  option(enable_units ${UNITS_DOC_STRING} OFF)
endif()

## clang-tidy
option(enable_clang_tidy "Run code through clang-tidy" OFF)

if(enable_clang_tidy)
  # Find clang-tidy
  find_program(CLANG_TIDY_BIN NAMES "clang-tidy" DOC "Path to clang-tidy executable")
  if(CLANG_TIDY_BIN)
    message(STATUS "clang-tidy found: ${CLANG_TIDY_BIN}")
    string(CONCAT CLANG_TIDY_CHECKS "-checks="
      "misc-static-assert,"
      "misc-unused-using-decls,"
      "misc-unused-parameters,"
      "misc-definitions-in-headers,"
      "misc-static-assert,"
      "modernize-use-override,"
      "modernize-deprecated-headers,"
      "modernize-pass-by-value,"
      "modernize-use-using,"
      "modernize-use-emplace,"
      "modernize-use-nullptr,"
      "modernize-use-equals-default,"
      "modernize-use-equals-delete,"
      "modernize-use-default-member-init,"
      "modernize-use-auto,"
      "modernize-raw-string-literal,"
      "modernize-make-shared,"
      "modernize-make-unique,"
      "modernize-loop-convert,"
      "readability-mismatched-tags,"
      "readability-make-member-function-const")
    
    set(CMAKE_CXX_CLANG_TIDY "${CLANG_TIDY_BIN}" "-fix" ${CLANG_TIDY_CHECKS} "-header-filter=^.*[^(pb)]\\.h$")
  else()
    message(FATAL_ERROR "clang-tidy not found")
  endif()
endif()


# Directory-scoped: applies to every target in the project (dccl,
# sub-libraries, tools, tests) without leaking to downstream consumers.
set(DCCL_DEBUG_OPTIONS -Wall -Wno-sign-compare -Wno-unknown-warning-option)
add_compile_options("$<$<CONFIG:DEBUG>:${DCCL_DEBUG_OPTIONS}>")
add_compile_options("$<$<CXX_COMPILER_ID:Clang>:-Wdocumentation>")

## configure to build/include/dccl
file(GLOB_RECURSE COPY_INCLUDE_FILES RELATIVE ${dccl_SRC_DIR} src/*.h src/*.proto src/*.hpp)
foreach(I ${COPY_INCLUDE_FILES})
  configure_file(${dccl_SRC_DIR}/${I} ${dccl_INC_DIR}/dccl/${I} @ONLY)
endforeach()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/dccl.h ${dccl_INC_DIR}/dccl.h @ONLY)

configure_file(${dccl_PYTHON_DIR}/setup.py.in ${dccl_PYTHON_DIR}/setup.py @ONLY)


## copy to build/share
file(GLOB SHARE_FILES RELATIVE ${dccl_SRC_DIR} src/share/*)
foreach(I ${SHARE_FILES})
  file(COPY ${dccl_SRC_DIR}/${I} DESTINATION ${dccl_SHARE_DIR})
endforeach()

## start adding subdirectories
add_subdirectory(src)

# Generate the package config file last, after all options/variables are set.
include(CMakePackageConfigHelpers)
configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/dccl-config.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/dccl-config.cmake"
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/dccl
)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/dccl-config.cmake"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/dccl
)
