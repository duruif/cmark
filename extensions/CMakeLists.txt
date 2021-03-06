cmake_minimum_required(VERSION 2.8)
set(LIBRARY "libcmark-gfmextensions")
set(STATICLIBRARY "libcmark-gfmextensions_static")
set(LIBRARY_SOURCES
    core-extensions.c
    table.c
    strikethrough.c
    autolink.c
    tagfilter.c
    ext_scanners.c
    ext_scanners.re
    ext_scanners.h
   )

include_directories(
  ${PROJECT_SOURCE_DIR}/src
  ${PROJECT_BINARY_DIR}/src
)

include (GenerateExportHeader)

# We make LIB_INSTALL_DIR configurable rather than
# hard-coding lib, because on some OSes different locations
# are used for different architectures (e.g. /usr/lib64 on
# 64-bit Fedora).
if(NOT LIB_INSTALL_DIR)
  set(LIB_INSTALL_DIR "lib" CACHE STRING
  "Set the installation directory for libraries." FORCE)
endif(NOT LIB_INSTALL_DIR)

include_directories(. ${CMAKE_CURRENT_BINARY_DIR})

set(CMAKE_C_FLAGS_PROFILE "${CMAKE_C_FLAGS_RELEASE} -pg")
set(CMAKE_LINKER_PROFILE "${CMAKE_LINKER_FLAGS_RELEASE} -pg")
add_compiler_export_flags()

if (CMARK_SHARED)
  add_library(${LIBRARY} SHARED ${LIBRARY_SOURCES})

  set_target_properties(${LIBRARY} PROPERTIES
  OUTPUT_NAME "cmark-gfmextensions"
  SOVERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}.gfm.${PROJECT_VERSION_GFM}
  VERSION ${PROJECT_VERSION})

  set_property(TARGET ${LIBRARY}
    APPEND PROPERTY MACOSX_RPATH true)

  # Avoid name clash between PROGRAM and LIBRARY pdb files.
  set_target_properties(${LIBRARY} PROPERTIES PDB_NAME cmark-gfmextensions_dll)

  list(APPEND CMARK_INSTALL ${LIBRARY})
  target_link_libraries(${LIBRARY} libcmark-gfm)

  generate_export_header(${LIBRARY} BASE_NAME cmarkextensions)
endif()

if (CMARK_STATIC)
  add_library(${STATICLIBRARY} STATIC ${LIBRARY_SOURCES})

  set_target_properties(${STATICLIBRARY} PROPERTIES
    COMPILE_FLAGS "-DCMARK_STATIC_DEFINE -DCMARKEXTENSIONS_STATIC_DEFINE"
    POSITION_INDEPENDENT_CODE ON)

  if (MSVC)
    set_target_properties(${STATICLIBRARY} PROPERTIES
      OUTPUT_NAME "cmark-gfmextensions_static"
      VERSION ${PROJECT_VERSION})
  else()
    set_target_properties(${STATICLIBRARY} PROPERTIES
      OUTPUT_NAME "cmark-gfmextensions"
      VERSION ${PROJECT_VERSION})
  endif(MSVC)
endif()

# Feature tests
include(CheckIncludeFile)
include(CheckCSourceCompiles)
include(CheckCSourceRuns)
include(CheckSymbolExists)
CHECK_INCLUDE_FILE(stdbool.h HAVE_STDBOOL_H)
CHECK_C_SOURCE_COMPILES(
  "int main() { __builtin_expect(0,0); return 0; }"
  HAVE___BUILTIN_EXPECT)
CHECK_C_SOURCE_COMPILES("
  int f(void) __attribute__ (());
  int main() { return 0; }
" HAVE___ATTRIBUTE__)

# Always compile with warnings
if(MSVC)
  # Force to always compile with W4
  if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
    string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W4")
  endif()
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /WX /wd4706 /wd4204 /wd4221 /wd4100 /D_CRT_SECURE_NO_WARNINGS")
elseif(CMAKE_COMPILER_IS_GNUCC OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wno-unused-parameter -std=c99 -pedantic")
endif()

# Compile as C++ under MSVC older than 12.0
if(MSVC AND MSVC_VERSION LESS 1800)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /TP")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Ubsan")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=undefined")
endif()
