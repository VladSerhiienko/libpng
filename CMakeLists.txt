
# CMakeLists.txt - CMake lists for libpng
#
# Copyright (c) 2018-2024 Cosmin Truta.
# Copyright (c) 2007-2018 Glenn Randers-Pehrson.
# Originally written by Christian Ehrlicher, 2007.
#
# Use, modification and distribution are subject to
# the same licensing terms and conditions as libpng.
# Please see the copyright notice in png.h or visit
# http://libpng.org/pub/png/src/libpng-LICENSE.txt
#
# For copyright and licensing purposes, please see
# the accompanying file scripts/cmake/AUTHORS.md
#
# SPDX-License-Identifier: libpng-2.0

cmake_minimum_required(VERSION 3.6)

set(PNGLIB_MAJOR 1)
set(PNGLIB_MINOR 6)
set(PNGLIB_REVISION 44)
set(PNGLIB_SUBREVISION "git")
set(PNGLIB_VERSION ${PNGLIB_MAJOR}.${PNGLIB_MINOR}.${PNGLIB_REVISION})
set(PNGLIB_ABI_VERSION ${PNGLIB_MAJOR}${PNGLIB_MINOR})
set(PNGLIB_SHARED_VERSION ${PNGLIB_ABI_VERSION}.${PNGLIB_REVISION}.${PNGLIB_SUBREVISION})

project(libpng
        VERSION ${PNGLIB_VERSION}
        LANGUAGES C ASM)

include(CheckCSourceCompiles)
include(GNUInstallDirs)

# Allow the users to specify an application-specific API prefix for libpng
# vendoring purposes. A standard libpng build should have no such prefix.
set(PNG_PREFIX ""
    CACHE STRING "Prefix to prepend to the API function names")

# Allow the users to import their own extra configuration settings.
set(DFA_XTRA ""
    CACHE FILEPATH "File containing extra configuration settings")

# Allow the users to switch on/off various library build types.
option(PNG_STATIC "Build libpng as a shared library" ON)

# Allow the users to configure various compilation options.
option(PNG_DEBUG "Enable debug output" OFF)
option(PNG_HARDWARE_OPTIMIZATIONS "Enable hardware optimizations" ON)

# Allow the users to specify an application-specific API prefix for libpng
# vendoring purposes. A standard libpng build should have no such prefix.
set(PNG_ZLIB_INCLUDE_DIRS ""
    CACHE STRING "Prefix to prepend to the API function names")

if(PNG_ZLIB_INCLUDE_DIRS STREQUAL "")
  message(SEND_ERROR "The option PNG_ZLIB_INCLUDE_DIRS is not set!")
endif()

message(STATUS "PNG_ZLIB_INCLUDE_DIRS=${PNG_ZLIB_INCLUDE_DIRS}")

if(UNIX AND NOT APPLE AND NOT BEOS AND NOT HAIKU AND NOT EMSCRIPTEN)
  find_library(M_LIBRARY m)
  if(M_LIBRARY)
    set(M_LIBRARY m)
  else()
    set(M_LIBRARY "")
  endif()
else()
  # libm is not available or not needed.
endif()

# CMake currently sets CMAKE_SYSTEM_PROCESSOR to one of x86_64 or arm64 on macOS,
# based upon the OS architecture, not the target architecture. As such, we need
# to check CMAKE_OSX_ARCHITECTURES to identify which hardware-specific flags to
# enable. Note that this will fail if you attempt to build a universal binary in
# a single CMake invocation.
if (APPLE AND CMAKE_OSX_ARCHITECTURES)
  set(TARGET_ARCH ${CMAKE_OSX_ARCHITECTURES})
else()
  set(TARGET_ARCH ${CMAKE_SYSTEM_PROCESSOR})
endif()

if(PNG_HARDWARE_OPTIMIZATIONS)

# Set definitions and sources for ARM.
if(TARGET_ARCH MATCHES "^(ARM|arm|aarch)")
  if(TARGET_ARCH MATCHES "^(ARM64|arm64|aarch64)")
    set(PNG_ARM_NEON_POSSIBLE_VALUES on off)
    set(PNG_ARM_NEON "on"
        CACHE STRING "Enable ARM NEON optimizations: on|off; on is default")
  else()
    set(PNG_ARM_NEON_POSSIBLE_VALUES check on off)
    set(PNG_ARM_NEON "off"
        CACHE STRING "Enable ARM NEON optimizations: check|on|off; off is default")
  endif()
  set_property(CACHE PNG_ARM_NEON
               PROPERTY STRINGS ${PNG_ARM_NEON_POSSIBLE_VALUES})
  list(FIND PNG_ARM_NEON_POSSIBLE_VALUES ${PNG_ARM_NEON} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_ARM_NEON must be one of [${PNG_ARM_NEON_POSSIBLE_VALUES}]")
  elseif(NOT PNG_ARM_NEON STREQUAL "off")
    set(libpng_arm_sources
        arm/arm_init.c
        arm/filter_neon_intrinsics.c
        arm/palette_neon_intrinsics.c)
    if(NOT MSVC)
      list(APPEND libpng_arm_sources arm/filter_neon.S)
    endif()
    if(PNG_ARM_NEON STREQUAL "on")
      add_definitions(-DPNG_ARM_NEON_OPT=2)
    elseif(PNG_ARM_NEON STREQUAL "check")
      add_definitions(-DPNG_ARM_NEON_CHECK_SUPPORTED)
    endif()
  else()
    add_definitions(-DPNG_ARM_NEON_OPT=0)
  endif()
endif()

# Set definitions and sources for PowerPC.
if(TARGET_ARCH MATCHES "^(powerpc|ppc64)")
  set(PNG_POWERPC_VSX_POSSIBLE_VALUES on off)
  set(PNG_POWERPC_VSX "on"
      CACHE STRING "Enable POWERPC VSX optimizations: on|off; on is default")
  set_property(CACHE PNG_POWERPC_VSX
               PROPERTY STRINGS ${PNG_POWERPC_VSX_POSSIBLE_VALUES})
  list(FIND PNG_POWERPC_VSX_POSSIBLE_VALUES ${PNG_POWERPC_VSX} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_POWERPC_VSX must be one of [${PNG_POWERPC_VSX_POSSIBLE_VALUES}]")
  elseif(NOT PNG_POWERPC_VSX STREQUAL "off")
    set(libpng_powerpc_sources
        powerpc/powerpc_init.c
        powerpc/filter_vsx_intrinsics.c)
    if(PNG_POWERPC_VSX STREQUAL "on")
      add_definitions(-DPNG_POWERPC_VSX_OPT=2)
    endif()
  else()
    add_definitions(-DPNG_POWERPC_VSX_OPT=0)
  endif()
endif()

# Set definitions and sources for Intel.
if(TARGET_ARCH MATCHES "^(i[3-6]86|x86|AMD64)")
  set(PNG_INTEL_SSE_POSSIBLE_VALUES on off)
  set(PNG_INTEL_SSE "on"
      CACHE STRING "Enable INTEL_SSE optimizations: on|off; on is default")
  set_property(CACHE PNG_INTEL_SSE
               PROPERTY STRINGS ${PNG_INTEL_SSE_POSSIBLE_VALUES})
  list(FIND PNG_INTEL_SSE_POSSIBLE_VALUES ${PNG_INTEL_SSE} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_INTEL_SSE must be one of [${PNG_INTEL_SSE_POSSIBLE_VALUES}]")
  elseif(NOT PNG_INTEL_SSE STREQUAL "off")
    set(libpng_intel_sources
        intel/intel_init.c
        intel/filter_sse2_intrinsics.c)
    if(PNG_INTEL_SSE STREQUAL "on")
      add_definitions(-DPNG_INTEL_SSE_OPT=1)
    endif()
  else()
    add_definitions(-DPNG_INTEL_SSE_OPT=0)
  endif()
endif()

# Set definitions and sources for MIPS.
if(TARGET_ARCH MATCHES "^(mipsel|mips64el)")
  set(PNG_MIPS_MSA_POSSIBLE_VALUES on off)
  set(PNG_MIPS_MSA "on"
      CACHE STRING "Enable MIPS_MSA optimizations: on|off; on is default")
  set_property(CACHE PNG_MIPS_MSA
               PROPERTY STRINGS ${PNG_MIPS_MSA_POSSIBLE_VALUES})
  list(FIND PNG_MIPS_MSA_POSSIBLE_VALUES ${PNG_MIPS_MSA} index_msa)
  if(index_msa EQUAL -1)
    message(FATAL_ERROR "PNG_MIPS_MSA must be one of [${PNG_MIPS_MSA_POSSIBLE_VALUES}]")
  endif()

  set(PNG_MIPS_MMI_POSSIBLE_VALUES on off)
  set(PNG_MIPS_MMI "on"
      CACHE STRING "Enable MIPS_MMI optimizations: on|off; on is default")
  set_property(CACHE PNG_MIPS_MMI
               PROPERTY STRINGS ${PNG_MIPS_MMI_POSSIBLE_VALUES})
  list(FIND PNG_MIPS_MMI_POSSIBLE_VALUES ${PNG_MIPS_MMI} index_mmi)
  if(index_mmi EQUAL -1)
    message(FATAL_ERROR "PNG_MIPS_MMI must be one of [${PNG_MIPS_MMI_POSSIBLE_VALUES}]")
  endif()

  if(PNG_MIPS_MSA STREQUAL "on" AND PNG_MIPS_MMI STREQUAL "on")
    set(libpng_mips_sources
        mips/mips_init.c
        mips/filter_msa_intrinsics.c
        mips/filter_mmi_inline_assembly.c)
    add_definitions(-DPNG_MIPS_MSA_OPT=2)
    add_definitions(-DPNG_MIPS_MMI_OPT=1)
  elseif(PNG_MIPS_MSA STREQUAL "on")
    set(libpng_mips_sources
        mips/mips_init.c
        mips/filter_msa_intrinsics.c)
    add_definitions(-DPNG_MIPS_MSA_OPT=2)
    add_definitions(-DPNG_MIPS_MMI_OPT=0)
  elseif(PNG_MIPS_MMI STREQUAL "on")
    set(libpng_mips_sources
        mips/mips_init.c
        mips/filter_mmi_inline_assembly.c)
    add_definitions(-DPNG_MIPS_MSA_OPT=0)
    add_definitions(-DPNG_MIPS_MMI_OPT=1)
    else()
    add_definitions(-DPNG_MIPS_MSA_OPT=0)
    add_definitions(-DPNG_MIPS_MMI_OPT=0)
    endif()
endif()

# Set definitions and sources for LoongArch.
if(TARGET_ARCH MATCHES "^(loongarch)")
  include(CheckCCompilerFlag)
  set(PNG_LOONGARCH_LSX_POSSIBLE_VALUES on off)
  set(PNG_LOONGARCH_LSX "on"
      CACHE STRING "Enable LOONGARCH_LSX optimizations: on|off; on is default")
  set_property(CACHE PNG_LOONGARCH_LSX
               PROPERTY STRINGS ${PNG_LOONGARCH_LSX_POSSIBLE_VALUES})
  list(FIND PNG_LOONGARCH_LSX_POSSIBLE_VALUES ${PNG_LOONGARCH_LSX} index)
  if(index EQUAL -1)
    message(FATAL_ERROR "PNG_LOONGARCH_LSX must be one of [${PNG_LOONGARCH_LSX_POSSIBLE_VALUES}]")
  elseif(NOT PNG_LOONGARCH_LSX STREQUAL "off")
    CHECK_C_COMPILER_FLAG("-mlsx" COMPILER_SUPPORTS_LSX)
    if(COMPILER_SUPPORTS_LSX)
      set(libpng_loongarch_sources
          loongarch/loongarch_lsx_init.c
          loongarch/filter_lsx_intrinsics.c)
      set_source_files_properties(${libpng_loongarch_sources}
                                  PROPERTIES
                                  COMPILE_FLAGS "-mlsx")
      add_definitions(-DPNG_LOONGARCH_LSX_OPT=1)
    else()
      message(FATAL_ERROR "Compiler does not support -mlsx option")
    endif()
  else()
    add_definitions(-DPNG_LOONGARCH_LSX_OPT=0)
  endif()
endif()

else(PNG_HARDWARE_OPTIMIZATIONS)

# Set definitions and sources for ARM.
if(TARGET_ARCH MATCHES "^(ARM|arm|aarch)")
  add_definitions(-DPNG_ARM_NEON_OPT=0)
endif()

# Set definitions and sources for PowerPC.
if(TARGET_ARCH MATCHES "^(powerpc|ppc64)")
  add_definitions(-DPNG_POWERPC_VSX_OPT=0)
endif()

# Set definitions and sources for Intel.
if(TARGET_ARCH MATCHES "^(i[3-6]86|x86|AMD64)")
  add_definitions(-DPNG_INTEL_SSE_OPT=0)
endif()

# Set definitions and sources for MIPS.
if(TARGET_ARCH MATCHES "^(mipsel|mips64el)")
  add_definitions(-DPNG_MIPS_MSA_OPT=0)
endif()

# Set definitions and sources for LoongArch.
if(TARGET_ARCH MATCHES "^(loongarch)")
  add_definitions(-DPNG_LOONGARCH_LSX_OPT=0)
endif()

endif(PNG_HARDWARE_OPTIMIZATIONS)

option(ld-version-script "Enable linker version script" ON)
if(ld-version-script AND NOT ANDROID AND NOT APPLE)
  # Check if LD supports linker scripts.
  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map" "
VERS_1 { global: sym1; local: *; };
VERS_2 { global: sym2; main; } VERS_1;
")
  set(_SAVED_CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS})
  if(NOT CMAKE_HOST_SOLARIS)
    # Avoid using CMAKE_SHARED_LIBRARY_C_FLAGS in version script checks on
    # Solaris, because of an incompatibility with the Solaris link editor.
    list(APPEND CMAKE_REQUIRED_FLAGS ${CMAKE_SHARED_LIBRARY_C_FLAGS})
  endif()
  list(APPEND CMAKE_REQUIRED_FLAGS "-Wl,--version-script='${CMAKE_CURRENT_BINARY_DIR}/conftest.map'")
  check_c_source_compiles("
void sym1(void) {}
void sym2(void) {}
int main(void) { return 0; }
" HAVE_LD_VERSION_SCRIPT)
  if(NOT HAVE_LD_VERSION_SCRIPT)
    set(CMAKE_REQUIRED_FLAGS ${_SAVED_CMAKE_REQUIRED_FLAGS})
    if(NOT CMAKE_HOST_SOLARIS)
      # Again, avoid using CMAKE_SHARED_LIBRARY_C_FLAGS in version script
      # checks on Solaris.
      list(APPEND CMAKE_REQUIRED_FLAGS ${CMAKE_SHARED_LIBRARY_C_FLAGS})
    endif()
    list(APPEND CMAKE_REQUIRED_FLAGS "-Wl,-M -Wl,${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
    check_c_source_compiles("
void sym1(void) {}
void sym2(void) {}
int main(void) { return 0; }
" HAVE_SOLARIS_LD_VERSION_SCRIPT)
  endif()
  set(CMAKE_REQUIRED_FLAGS ${_SAVED_CMAKE_REQUIRED_FLAGS})
  file(REMOVE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
endif()

include_directories(${CMAKE_CURRENT_BINARY_DIR})

# No awk available to generate sources; use pre-built pnglibconf.h
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/pnglibconf.h.prebuilt
                ${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h)
add_custom_target(png_genfiles)

# List the source code files.
set(libpng_public_hdrs
    png.h
    pngconf.h
    "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h"
)
set(libpng_private_hdrs
    pngpriv.h
    pngdebug.h
    pnginfo.h
    pngstruct.h
)

set(libpng_sources
    ${libpng_public_hdrs}
    ${libpng_private_hdrs}
    png.c
    pngerror.c
    pngget.c
    pngmem.c
    pngpread.c
    pngread.c
    pngrio.c
    pngrtran.c
    pngrutil.c
    pngset.c
    pngtrans.c
    pngwio.c
    pngwrite.c
    pngwtran.c
    pngwutil.c
    ${libpng_arm_sources}
    ${libpng_intel_sources}
    ${libpng_mips_sources}
    ${libpng_powerpc_sources}
    ${libpng_loongarch_sources}
)
set(pngtest_sources
    pngtest.c
)
set(pngvalid_sources
    contrib/libtests/pngvalid.c
)
set(pngstest_sources
    contrib/libtests/pngstest.c
)
set(pngunknown_sources
    contrib/libtests/pngunknown.c
)
set(pngimage_sources
    contrib/libtests/pngimage.c
)
set(pngfix_sources
    contrib/tools/pngfix.c
)
set(png_fix_itxt_sources
    contrib/tools/png-fix-itxt.c
)

if(MSVC OR (WIN32 AND (CMAKE_C_COMPILER_ID MATCHES "Clang")))
  add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
  add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
endif()

if(PNG_DEBUG)
  add_definitions(-DPNG_DEBUG)
endif()

# Now build our targets.

# Initialize the list of libpng library targets.
set(PNG_LIBRARY_TARGETS "")

# Initialize the libpng library file names.
if(UNIX
   OR (WIN32 AND NOT CMAKE_SHARED_LIBRARY_PREFIX STREQUAL "")
   OR (WIN32 AND NOT CMAKE_STATIC_LIBRARY_PREFIX STREQUAL ""))
  # We are on a Unix or Unix-like toolchain like the GNU toolchain on Windows.
  # Library file names are expected to have an implicit prefix such as "lib".
  # Let CMake prepend and append its usual prefixes and suffixes by default.
  set(PNG_SHARED_OUTPUT_NAME "png${PNGLIB_ABI_VERSION}")
  set(PNG_STATIC_OUTPUT_NAME "png${PNGLIB_ABI_VERSION}")
else()
  # We are, most likely, on a Windows toolchain like MSVC, Clang on Windows,
  # Borland/Embarcadero, etc. We need to specify the "libpng" name explicitly.
  # We also need to use a custom suffix, in order to distinguish between the
  # shared import library name and the static library name.
  set(PNG_SHARED_OUTPUT_NAME "libpng${PNGLIB_ABI_VERSION}")
  set(PNG_STATIC_OUTPUT_NAME "libpng${PNGLIB_ABI_VERSION}_static")
endif()

if(PNG_STATIC)
  add_library(png_static STATIC ${libpng_sources})
  add_dependencies(png_static png_genfiles)
  list(APPEND PNG_LIBRARY_TARGETS png_static)
  set_target_properties(png_static PROPERTIES
                        OUTPUT_NAME "${PNG_STATIC_OUTPUT_NAME}")
  target_include_directories(png_static
                             PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                             PRIVATE ${PNG_ZLIB_INCLUDE_DIRS})
  target_include_directories(png_static SYSTEM
                             INTERFACE $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/libpng${PNGLIB_ABI_VERSION}>)
  if (M_LIBRARY)
    target_link_libraries(png_static PUBLIC ${M_LIBRARY})
  endif()
endif()

if(NOT PNG_LIBRARY_TARGETS)
  message(SEND_ERROR "No library variant selected to build. "
                     "Please enable at least one of the following options: "
                     "PNG_SHARED, PNG_STATIC, PNG_FRAMEWORK")
endif()

# Create source generation scripts.
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/cmake/genchk.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/cmake/genchk.cmake
               @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/cmake/genout.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/cmake/genout.cmake
               @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/cmake/gensrc.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/cmake/gensrc.cmake
               @ONLY)
