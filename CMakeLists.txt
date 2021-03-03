cmake_minimum_required (VERSION 3.11)
project (validator-keys-tool)

#[===========================================[
  The tool depends on the xrpl_core
  library which is defined by the brtd
  project. This looks for installed brtd
  libs and, if not found, pulls them in with
  FetchContent.
#]===========================================]
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/Builds/CMake")
include(KeysNIH)

get_directory_property(has_parent PARENT_DIRECTORY)
if(coverage)
  # Rippled also responds to the "coverage" flag, so clear it if it's set until
  # the dependency is set up.
  set(keys_coverage ${coverage})
  set(coverage OFF CACHE BOOL "gcc/clang only" FORCE)
endif()
if (NOT has_parent)
  find_package(Ripple QUIET)
  if (NOT TARGET Ripple::xrpl_core)
    find_package(Git)

    if (NOT GIT_FOUND)
      message (FATAL_ERROR "git is required to determine branch name")
    endif ()

    execute_process (COMMAND ${GIT_EXECUTABLE} "rev-parse" "--abbrev-ref" "HEAD"
                     WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
                     RESULT_VARIABLE _git_exit_code
                     OUTPUT_VARIABLE _branch
                     OUTPUT_STRIP_TRAILING_WHITESPACE
                     ERROR_QUIET)
    if (NOT _git_exit_code EQUAL 0)
      message (WARNING "git command failed - deduced branch might be incorrect")
    endif ()

    # brtd_tag is cache string and can be overriden when configuring
    # with -Dbrtd_tag=commit_or_tag in order to pick a specific
    # brtd version to download. Default tag is develop/master/release as 
    # determined by the branch of this project
    if (NOT (_branch STREQUAL "master" OR _branch STREQUAL "release"))
      set (brtd_tag "develop" CACHE STRING
        "tag/commit of brtd to fetch from if a local install is not found")
    else ()
      set (brtd_tag "${_branch}" CACHE STRING
        "tag/commit of brtd to fetch from if a local install is not found")
    endif ()
    message(STATUS "Installed brtd not found... \
      using local copy from tag/commit [${brtd_tag}]")
    include (FetchContent)
    FetchContent_Declare(
      brtd_src
      GIT_REPOSITORY https://github.com/BRTNetwork/brtd.git
      GIT_TAG ${brtd_tag}
    )
    FetchContent_GetProperties(brtd_src)
    if(NOT brtd_src_POPULATED)
      message (STATUS "Pausing to download brtd source...")
      FetchContent_Populate(brtd_src)
      add_subdirectory(${brtd_src_SOURCE_DIR} ${brtd_src_BINARY_DIR})
    endif()
  endif ()
endif ()
if(keys_coverage)
  set(coverage ${keys_coverage} CACHE BOOL "gcc/clang only" FORCE)
endif()
include(KeysSanity)
include(KeysCov)
include(KeysInterface)

add_executable (validator-keys
  src/ValidatorKeys.cpp
  src/ValidatorKeysTool.cpp
  ## UNIT TESTS:
  src/test/ValidatorKeys_test.cpp
  src/test/ValidatorKeysTool_test.cpp)
target_include_directories (validator-keys PRIVATE src)
target_link_libraries (validator-keys Ripple::xrpl_core Keys::opts)

if (has_parent)
  set_target_properties (validator-keys PROPERTIES EXCLUDE_FROM_ALL ON)
  set_target_properties (validator-keys PROPERTIES EXCLUDE_FROM_DEFAULT_BUILD ON)
endif ()



