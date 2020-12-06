# ----------------------------------------------------------------------------
#  Root CMake file for OpenCV
#
#    From the off-tree build directory, invoke:
#      $ cmake <PATH_TO_OPENCV_ROOT>
#
# ----------------------------------------------------------------------------

# Disable in-source builds to prevent source tree corruption.
if(" ${CMAKE_SOURCE_DIR}" STREQUAL " ${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "
FATAL: In-source builds are not allowed.
       You should create a separate directory for build files.
")
endif()


include(cmake/OpenCVMinDepVersions.cmake)

if(CMAKE_GENERATOR MATCHES Xcode AND XCODE_VERSION VERSION_GREATER 4.3)
  cmake_minimum_required(VERSION 3.0 FATAL_ERROR)
elseif(CMAKE_SYSTEM_NAME MATCHES WindowsPhone OR CMAKE_SYSTEM_NAME MATCHES WindowsStore)
  cmake_minimum_required(VERSION 3.1 FATAL_ERROR)
  #Required to resolve linker error issues due to incompatibility with CMake v3.0+ policies.
  #CMake fails to find _fseeko() which leads to subsequent linker error.
  #See details here: http://www.cmake.org/Wiki/CMake/Policies
  cmake_policy(VERSION 2.8)
else()
  cmake_minimum_required(VERSION "${MIN_VER_CMAKE}" FATAL_ERROR)
endif()

#
# Configure CMake policies
#
if(POLICY CMP0026)
  cmake_policy(SET CMP0026 NEW)
endif()

if(POLICY CMP0042)
  cmake_policy(SET CMP0042 NEW)  # CMake 3.0+ (2.8.12): MacOS "@rpath" in target's install name
endif()

if(POLICY CMP0046)
  cmake_policy(SET CMP0046 NEW)  # warn about non-existed dependencies
endif()

if(POLICY CMP0051)
  cmake_policy(SET CMP0051 NEW)
endif()

if(POLICY CMP0054)  # CMake 3.1: Only interpret if() arguments as variables or keywords when unquoted.
  cmake_policy(SET CMP0054 NEW)
endif()

if(POLICY CMP0056)
  cmake_policy(SET CMP0056 NEW)  # try_compile(): link flags
endif()

if(POLICY CMP0066)
  cmake_policy(SET CMP0066 NEW)  # CMake 3.7: try_compile(): use per-config flags, like CMAKE_CXX_FLAGS_RELEASE
endif()

if(POLICY CMP0067)
  cmake_policy(SET CMP0067 NEW)  # CMake 3.8: try_compile(): honor language standard variables (like C++11)
endif()

if(POLICY CMP0068)
  cmake_policy(SET CMP0068 NEW)  # CMake 3.9+: `RPATH` settings on macOS do not affect `install_name`.
endif()

if(POLICY CMP0075)
  cmake_policy(SET CMP0075 NEW)  # CMake 3.12+: Include file check macros honor `CMAKE_REQUIRED_LIBRARIES`
endif()

if(POLICY CMP0077)
  cmake_policy(SET CMP0077 NEW)  # CMake 3.13+: option() honors normal variables.
endif()

#
# Configure OpenCV CMake hooks
#
set(CMAKE_CXX_FLAGS_RELEASE "-Os")
set(CMAKE_C_FLAGS_RELEASE "-Os")

include(cmake/OpenCVUtils.cmake)
ocv_cmake_reset_hooks()
ocv_check_environment_variables(OPENCV_CMAKE_HOOKS_DIR)
if(DEFINED OPENCV_CMAKE_HOOKS_DIR)
  foreach(__dir ${OPENCV_CMAKE_HOOKS_DIR})
    get_filename_component(__dir "${__dir}" ABSOLUTE)
    ocv_cmake_hook_register_dir(${__dir})
  endforeach()
endif()

ocv_cmake_hook(CMAKE_INIT)

# must go before the project()/enable_language() commands
ocv_update(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "Configs" FORCE)
if(DEFINED CMAKE_BUILD_TYPE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "${CMAKE_CONFIGURATION_TYPES}")
endif()

option(ENABLE_PIC "Generate position independent code (necessary for shared libraries)" TRUE)
set(CMAKE_POSITION_INDEPENDENT_CODE ${ENABLE_PIC})

ocv_cmake_hook(PRE_CMAKE_BOOTSTRAP)

# Bootstap CMake system: setup CMAKE_SYSTEM_NAME and other vars
if(OPENCV_WORKAROUND_CMAKE_20989)
  set(CMAKE_SYSTEM_PROCESSOR_BACKUP ${CMAKE_SYSTEM_PROCESSOR})
endif()
enable_language(CXX C)
if(OPENCV_WORKAROUND_CMAKE_20989)
  set(CMAKE_SYSTEM_PROCESSOR ${CMAKE_SYSTEM_PROCESSOR_BACKUP})
endif()

ocv_cmake_hook(POST_CMAKE_BOOTSTRAP)

if(NOT OPENCV_SKIP_CMAKE_SYSTEM_FILE)
  include("cmake/platforms/OpenCV-${CMAKE_SYSTEM_NAME}.cmake" OPTIONAL RESULT_VARIABLE "OPENCV_CMAKE_SYSTEM_FILE")
  if(NOT OPENCV_CMAKE_SYSTEM_FILE)
    message(STATUS "OpenCV: system-specific configuration file is not found: '${CMAKE_SYSTEM_NAME}'")
  endif()
endif()

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)  # https://cmake.org/cmake/help/latest/variable/CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT.html
  if(NOT CMAKE_TOOLCHAIN_FILE)
    if(WIN32)
      set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "Installation Directory" FORCE)
    else()
      set(CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "Installation Directory" FORCE)
    endif()
  else()
    # any cross-compiling
    set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "Installation Directory" FORCE)
  endif()
endif()

enable_testing()

project(OpenCV CXX C)

if(MSVC)
  set(CMAKE_USE_RELATIVE_PATHS ON CACHE INTERNAL "" FORCE)
endif()

ocv_cmake_eval(DEBUG_PRE ONCE)

ocv_clear_vars(OpenCVModules_TARGETS)

include(cmake/OpenCVDownload.cmake)

set(BUILD_LIST "" CACHE STRING "Build only listed modules (comma-separated, e.g. 'videoio,dnn,ts')")

# ----------------------------------------------------------------------------
# Break in case of popular CMake configuration mistakes
# ----------------------------------------------------------------------------
if(NOT CMAKE_SIZEOF_VOID_P GREATER 0)
  message(FATAL_ERROR "CMake fails to determine the bitness of the target platform.
  Please check your CMake and compiler installation. If you are cross-compiling then ensure that your CMake toolchain file correctly sets the compiler details.")
endif()

# ----------------------------------------------------------------------------
# Detect compiler and target platform architecture
# ----------------------------------------------------------------------------
include(cmake/OpenCVDetectCXXCompiler.cmake)
ocv_cmake_hook(POST_DETECT_COMPILER)

# Add these standard paths to the search paths for FIND_LIBRARY
# to find libraries from these locations first
if(UNIX AND NOT ANDROID)
  if(X86_64 OR CMAKE_SIZEOF_VOID_P EQUAL 8)
    if(EXISTS /lib64)
      list(APPEND CMAKE_LIBRARY_PATH /lib64)
    else()
      list(APPEND CMAKE_LIBRARY_PATH /lib)
    endif()
    if(EXISTS /usr/lib64)
      list(APPEND CMAKE_LIBRARY_PATH /usr/lib64)
    else()
      list(APPEND CMAKE_LIBRARY_PATH /usr/lib)
    endif()
  elseif(X86 OR CMAKE_SIZEOF_VOID_P EQUAL 4)
    if(EXISTS /lib32)
      list(APPEND CMAKE_LIBRARY_PATH /lib32)
    else()
      list(APPEND CMAKE_LIBRARY_PATH /lib)
    endif()
    if(EXISTS /usr/lib32)
      list(APPEND CMAKE_LIBRARY_PATH /usr/lib32)
    else()
      list(APPEND CMAKE_LIBRARY_PATH /usr/lib)
    endif()
  endif()
endif()

# Add these standard paths to the search paths for FIND_PATH
# to find include files from these locations first
if(MINGW)
  if(EXISTS /mingw)
      list(APPEND CMAKE_INCLUDE_PATH /mingw)
  endif()
  if(EXISTS /mingw32)
      list(APPEND CMAKE_INCLUDE_PATH /mingw32)
  endif()
  if(EXISTS /mingw64)
      list(APPEND CMAKE_INCLUDE_PATH /mingw64)
  endif()
endif()

# ----------------------------------------------------------------------------
# OpenCV cmake options
# ----------------------------------------------------------------------------

# Optional 3rd party components
# ===================================================

# OpenCV build components
# ===================================================
OCV_OPTION(BUILD_PACKAGE            "Enables 'make package_source' command"       ON  IF NOT WINRT)
OCV_OPTION(BUILD_OBJC               "Enable Objective-C support"                  ON  IF APPLE_FRAMEWORK )

# OpenCV installation options - tingchiun
# ===================================================
OCV_OPTION(INSTALL_CREATE_DISTRIB   "Change install rules to build the distribution package" OFF )
OCV_OPTION(INSTALL_BIN_EXAMPLES     "Install prebuilt examples" WIN32 IF BUILD_EXAMPLES)
OCV_OPTION(INSTALL_C_EXAMPLES       "Install C examples"        OFF )
OCV_OPTION(INSTALL_PYTHON_EXAMPLES  "Install Python examples"   OFF )
OCV_OPTION(INSTALL_ANDROID_EXAMPLES "Install Android examples"  OFF IF ANDROID )
OCV_OPTION(INSTALL_TO_MANGLED_PATHS "Enables mangled install paths, that help with side by side installs." OFF IF (UNIX AND NOT ANDROID AND NOT APPLE_FRAMEWORK AND BUILD_SHARED_LIBS) )
OCV_OPTION(INSTALL_TESTS            "Install accuracy and performance test binaries and test data" OFF)

# OpenCV build options - disable these all have -11MB downsize
# ===================================================
#OCV_OPTION(ENABLE_CCACHE              "Use ccache"                                               (UNIX AND NOT IOS AND (CMAKE_GENERATOR MATCHES "Makefile" OR CMAKE_GENERATOR MATCHES "Ninja")) )
#OCV_OPTION(ENABLE_PRECOMPILED_HEADERS "Use precompiled headers"                                  MSVC IF (MSVC OR (NOT IOS AND NOT CMAKE_CROSSCOMPILING) ) )
#OCV_OPTION(ENABLE_SOLUTION_FOLDERS    "Solution folder in Visual Studio or in other IDEs"        (MSVC_IDE OR CMAKE_GENERATOR MATCHES Xcode) )
#OCV_OPTION(ENABLE_PROFILING           "Enable profiling in the GCC compiler (Add flags: -g -pg)" OFF  IF CV_GCC )
#OCV_OPTION(ENABLE_COVERAGE            "Enable coverage collection with  GCov"                    OFF  IF CV_GCC )
#OCV_OPTION(OPENCV_ENABLE_MEMORY_SANITIZER "Better support for memory/address sanitizers"         OFF)
#OCV_OPTION(ENABLE_OMIT_FRAME_POINTER  "Enable -fomit-frame-pointer for GCC"                      ON   IF CV_GCC )
#OCV_OPTION(ENABLE_POWERPC             "Enable PowerPC for GCC"                                   ON   IF (CV_GCC AND CMAKE_SYSTEM_PROCESSOR MATCHES powerpc.*) )
#OCV_OPTION(ENABLE_FAST_MATH           "Enable compiler options for fast math optimizations on FP computations (not recommended)" OFF)
#if(NOT IOS AND (NOT ANDROID OR OPENCV_ANDROID_USE_LEGACY_FLAGS))  # Use CPU_BASELINE instead
#OCV_OPTION(ENABLE_NEON                "Enable NEON instructions"                                 (NEON OR ANDROID_ARM_NEON OR AARCH64) IF (CV_GCC OR CV_CLANG) AND (ARM OR AARCH64 OR IOS) )
#OCV_OPTION(ENABLE_VFPV3               "Enable VFPv3-D32 instructions"                            OFF  IF (CV_GCC OR CV_CLANG) AND (ARM OR AARCH64 OR IOS) )
#endif()
#OCV_OPTION(ENABLE_NOISY_WARNINGS      "Show all warnings even if they are too noisy"             OFF )
#OCV_OPTION(OPENCV_WARNINGS_ARE_ERRORS "Treat warnings as errors"                                 OFF )
#OCV_OPTION(ANDROID_EXAMPLES_WITH_LIBS "Build binaries of Android examples with native libraries" OFF  IF ANDROID )
#OCV_OPTION(ENABLE_IMPL_COLLECTION     "Collect implementation data on function call"             OFF )
#OCV_OPTION(ENABLE_INSTRUMENTATION     "Instrument functions to collect calls trace and performance" OFF )
#OCV_OPTION(ENABLE_GNU_STL_DEBUG       "Enable GNU STL Debug mode (defines _GLIBCXX_DEBUG)"       OFF IF ((NOT CMAKE_VERSION VERSION_LESS "2.8.11") AND CV_GCC) )
#OCV_OPTION(ENABLE_BUILD_HARDENING     "Enable hardening of the resulting binaries (against security attacks, detects memory corruption, etc)" OFF)
#OCV_OPTION(ENABLE_LTO                 "Enable Link Time Optimization" OFF IF CV_GCC OR MSVC)
#OCV_OPTION(ENABLE_THIN_LTO            "Enable Thin LTO" OFF IF CV_CLANG)
#OCV_OPTION(GENERATE_ABI_DESCRIPTOR    "Generate XML file for abi_compliance_checker tool" OFF IF UNIX)
#OCV_OPTION(OPENCV_GENERATE_PKGCONFIG  "Generate .pc file for pkg-config build tool (deprecated)" OFF)
#OCV_OPTION(CV_ENABLE_INTRINSICS       "Use intrinsic-based optimized code" ON )
#OCV_OPTION(CV_DISABLE_OPTIMIZATION    "Disable explicit optimized code (dispatched code/intrinsics/loop unrolling/etc)" OFF )
#OCV_OPTION(CV_TRACE                   "Enable OpenCV code trace" ON)
#OCV_OPTION(OPENCV_GENERATE_SETUPVARS  "Generate setup_vars* scripts" ON IF (NOT ANDROID AND NOT APPLE_FRAMEWORK) )
#OCV_OPTION(ENABLE_CONFIG_VERIFICATION "Fail build if actual configuration doesn't match requested (WITH_XXX != HAVE_XXX)" OFF)
#OCV_OPTION(OPENCV_ENABLE_MEMALIGN     "Enable posix_memalign or memalign usage" ON)

#OCV_OPTION(ENABLE_PYLINT              "Add target with Pylint checks"                            (BUILD_DOCS OR BUILD_EXAMPLES) IF (NOT CMAKE_CROSSCOMPILING AND NOT APPLE_FRAMEWORK) )
#OCV_OPTION(ENABLE_FLAKE8              "Add target with Python flake8 checker"                    (BUILD_DOCS OR BUILD_EXAMPLES) IF (NOT CMAKE_CROSSCOMPILING AND NOT APPLE_FRAMEWORK) )

#if(ENABLE_IMPL_COLLECTION)
#  add_definitions(-DCV_COLLECT_IMPL_DATA)
#endif()

#set(OPENCV_MATHJAX_RELPATH "https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.0" CACHE STRING "URI to a MathJax installation")

# ----------------------------------------------------------------------------
#  Get actual OpenCV version number from sources
# ----------------------------------------------------------------------------
include(cmake/OpenCVVersion.cmake)

ocv_cmake_hook(POST_OPTIONS)

# ----------------------------------------------------------------------------
#  Build & install layouts before 105.5 - after 105.5
# ----------------------------------------------------------------------------

#if(OPENCV_TEST_DATA_PATH)
#  get_filename_component(OPENCV_TEST_DATA_PATH ${OPENCV_TEST_DATA_PATH} ABSOLUTE)
#endif()

# Save libs and executables in the same place
set(EXECUTABLE_OUTPUT_PATH "${CMAKE_BINARY_DIR}/bin" CACHE PATH "Output directory for applications")

if(ANDROID)
  set(LIBRARY_OUTPUT_PATH                "${OpenCV_BINARY_DIR}/lib/${ANDROID_NDK_ABI_NAME}")
  ocv_update(3P_LIBRARY_OUTPUT_PATH      "${OpenCV_BINARY_DIR}/3rdparty/lib/${ANDROID_NDK_ABI_NAME}")
else()
  set(LIBRARY_OUTPUT_PATH                "${OpenCV_BINARY_DIR}/lib")
  #ocv_update(3P_LIBRARY_OUTPUT_PATH      "${OpenCV_BINARY_DIR}/3rdparty/lib")
endif()

if(ANDROID)
  if(ANDROID_ABI MATCHES "NEON")
    set(ENABLE_NEON ON)
  endif()
  if(ANDROID_ABI MATCHES "VFPV3")
    set(ENABLE_VFPV3 ON)
  endif()
endif()

if(WIN32)
  # Postfix of DLLs:
  ocv_update(OPENCV_DLLVERSION "${OPENCV_VERSION_MAJOR}${OPENCV_VERSION_MINOR}${OPENCV_VERSION_PATCH}")
  ocv_update(OPENCV_DEBUG_POSTFIX d)
else()
  # Postfix of so's:
  ocv_update(OPENCV_DLLVERSION "")
  ocv_update(OPENCV_DEBUG_POSTFIX "")
endif()

#if(DEFINED CMAKE_DEBUG_POSTFIX)
#  set(OPENCV_DEBUG_POSTFIX "${CMAKE_DEBUG_POSTFIX}")
#endif()

#if((INSTALL_CREATE_DISTRIB AND BUILD_SHARED_LIBS AND NOT DEFINED BUILD_opencv_world) OR APPLE_FRAMEWORK)
#  set(BUILD_opencv_world ON CACHE INTERNAL "")
#endif()

include(cmake/OpenCVInstallLayout.cmake)

# ----------------------------------------------------------------------------
#  Path for build/platform -specific headers
# ----------------------------------------------------------------------------
ocv_update(OPENCV_CONFIG_FILE_INCLUDE_DIR "${CMAKE_BINARY_DIR}/" CACHE PATH "Where to create the platform-dependant cvconfig.h")
ocv_include_directories(${OPENCV_CONFIG_FILE_INCLUDE_DIR})

# ----------------------------------------------------------------------------
#  Path for additional modules - before 105.5 after 105.5
# ----------------------------------------------------------------------------
#set(OPENCV_EXTRA_MODULES_PATH "" CACHE PATH "Where to look for additional OpenCV modules (can be ;-separated list of paths)")

# ----------------------------------------------------------------------------
#  Autodetect if we are in a GIT repository
# ----------------------------------------------------------------------------
find_host_package(Git QUIET)

if(NOT DEFINED OPENCV_VCSVERSION AND GIT_FOUND)
  ocv_git_describe(OPENCV_VCSVERSION "${OpenCV_SOURCE_DIR}")
elseif(NOT DEFINED OPENCV_VCSVERSION)
  # We don't have git:
  set(OPENCV_VCSVERSION "unknown")
endif()


# ----------------------------------------------------------------------------
# OpenCV compiler and linker options
# ----------------------------------------------------------------------------
# In case of Makefiles if the user does not setup CMAKE_BUILD_TYPE, assume it's Release:
if(CMAKE_GENERATOR MATCHES "Makefiles|Ninja" AND "${CMAKE_BUILD_TYPE}" STREQUAL "")
  set(CMAKE_BUILD_TYPE Release)
endif()

ocv_cmake_hook(POST_CMAKE_BUILD_OPTIONS)

# --- Python Support ---
if(NOT IOS)
  include(cmake/OpenCVDetectPython.cmake)
endif()

include(cmake/OpenCVCompilerOptions.cmake)

ocv_cmake_hook(POST_COMPILER_OPTIONS)

# ----------------------------------------------------------------------------
#       CHECK FOR SYSTEM LIBRARIES, OPTIONS, ETC..
# ----------------------------------------------------------------------------
if(UNIX)
  if(NOT APPLE_FRAMEWORK OR OPENCV_ENABLE_PKG_CONFIG)
    if(CMAKE_CROSSCOMPILING AND NOT DEFINED ENV{PKG_CONFIG_LIBDIR} AND NOT DEFINED ENV{PKG_CONFIG_SYSROOT_DIR}
        AND NOT OPENCV_ENABLE_PKG_CONFIG
    )
      if(NOT PkgConfig_FOUND)
        message(STATUS "OpenCV disables pkg-config to avoid using of host libraries. Consider using PKG_CONFIG_LIBDIR to specify target SYSROOT")
      elseif(OPENCV_SKIP_PKG_CONFIG_WARNING)
        message(WARNING "pkg-config is enabled in cross-compilation mode without defining of PKG_CONFIG_LIBDIR environment variable. This may lead to misconfigured host-based dependencies.")
      endif()
    elseif(OPENCV_DISABLE_PKG_CONFIG)
      if(PkgConfig_FOUND)
        message(WARNING "OPENCV_DISABLE_PKG_CONFIG flag has no effect")
      endif()
    else()
      find_package(PkgConfig QUIET)
    endif()
  endif()
  include(CheckFunctionExists)
  include(CheckIncludeFile)
  include(CheckSymbolExists)

  if(NOT APPLE)
    CHECK_INCLUDE_FILE(pthread.h HAVE_PTHREAD)
    if(ANDROID)
      set(OPENCV_LINKER_LIBS ${OPENCV_LINKER_LIBS} dl m log)
    elseif(CMAKE_SYSTEM_NAME MATCHES "FreeBSD|NetBSD|DragonFly|OpenBSD|Haiku")
      set(OPENCV_LINKER_LIBS ${OPENCV_LINKER_LIBS} m pthread)
    elseif(EMSCRIPTEN)
      # no need to link to system libs with emscripten
    else()
      set(OPENCV_LINKER_LIBS ${OPENCV_LINKER_LIBS} dl m pthread rt)
    endif()
  else()
    set(HAVE_PTHREAD 1)
  endif()

  if(OPENCV_ENABLE_MEMALIGN)
    CHECK_SYMBOL_EXISTS(posix_memalign stdlib.h HAVE_POSIX_MEMALIGN)
    CHECK_INCLUDE_FILE(malloc.h HAVE_MALLOC_H)
    if(HAVE_MALLOC_H)
      CHECK_SYMBOL_EXISTS(memalign malloc.h HAVE_MEMALIGN)
    endif()
    # TODO:
    # - _aligned_malloc() on Win32
    # - std::aligned_alloc() C++17 / C11
  endif()
endif()

include(cmake/OpenCVPCHSupport.cmake)
include(cmake/OpenCVModule.cmake)

# ----------------------------------------------------------------------------
#  Detect endianness of build platform - before 105.5 - after 105.5
# ----------------------------------------------------------------------------
if(IOS)
  # test_big_endian needs try_compile, which doesn't work for iOS
  # http://public.kitware.com/Bug/view.php?id=12288
  set(WORDS_BIGENDIAN 0)
else()
  include(TestBigEndian)
  test_big_endian(WORDS_BIGENDIAN)
endif()



# ----------------------------------------------------------------------------
#  Detect 3rd-party libraries  - before 109.5 - after 105.5
# ----------------------------------------------------------------------------


# ----------------------------------------------------------------------------
# OpenCV HAL
# ----------------------------------------------------------------------------
set(_hal_includes "")
macro(ocv_hal_register HAL_LIBRARIES_VAR HAL_HEADERS_VAR HAL_INCLUDE_DIRS_VAR)
  # 1. libraries
  foreach (l ${${HAL_LIBRARIES_VAR}})
    if(NOT TARGET ${l})
      get_filename_component(l "${l}" ABSOLUTE)
    endif()
    list(APPEND OPENCV_HAL_LINKER_LIBS ${l})
  endforeach()
  # 2. headers
  foreach (h ${${HAL_HEADERS_VAR}})
    set(_hal_includes "${_hal_includes}\n#include \"${h}\"")
  endforeach()
  # 3. include paths
  ocv_include_directories(${${HAL_INCLUDE_DIRS_VAR}})
endmacro()

if(NOT DEFINED OpenCV_HAL)
  set(OpenCV_HAL "OpenCV_HAL")
endif()

if(HAVE_OPENVX)
  if(NOT ";${OpenCV_HAL};" MATCHES ";openvx;")
    set(OpenCV_HAL "openvx;${OpenCV_HAL}")
  endif()
endif()

if(WITH_CAROTENE)
  ocv_debug_message(STATUS "Enable carotene acceleration")
  if(NOT ";${OpenCV_HAL};" MATCHES ";carotene;")
    set(OpenCV_HAL "carotene;${OpenCV_HAL}")
  endif()
endif()

foreach(hal ${OpenCV_HAL})
  if(hal STREQUAL "carotene")
    if(";${CPU_BASELINE_FINAL};" MATCHES ";NEON;")
      add_subdirectory(3rdparty/carotene/hal)
      ocv_hal_register(CAROTENE_HAL_LIBRARIES CAROTENE_HAL_HEADERS CAROTENE_HAL_INCLUDE_DIRS)
      list(APPEND OpenCV_USED_HAL "carotene (ver ${CAROTENE_HAL_VERSION})")
    else()
      message(STATUS "Carotene: NEON is not available, disabling carotene...")
    endif()
  elseif(hal STREQUAL "openvx")
    add_subdirectory(3rdparty/openvx)
    ocv_hal_register(OPENVX_HAL_LIBRARIES OPENVX_HAL_HEADERS OPENVX_HAL_INCLUDE_DIRS)
    list(APPEND OpenCV_USED_HAL "openvx (ver ${OPENVX_HAL_VERSION})")
  else()
    ocv_debug_message(STATUS "OpenCV HAL: ${hal} ...")
    ocv_clear_vars(OpenCV_HAL_LIBRARIES OpenCV_HAL_HEADERS OpenCV_HAL_INCLUDE_DIRS)
    find_package(${hal} NO_MODULE QUIET)
    if(${hal}_FOUND)
      ocv_hal_register(OpenCV_HAL_LIBRARIES OpenCV_HAL_HEADERS OpenCV_HAL_INCLUDE_DIRS)
      list(APPEND OpenCV_USED_HAL "${hal} (ver ${${hal}_VERSION})")
    endif()
  endif()
endforeach()
configure_file("${OpenCV_SOURCE_DIR}/cmake/templates/custom_hal.hpp.in" "${CMAKE_BINARY_DIR}/custom_hal.hpp" @ONLY)
unset(_hal_includes)


# ----------------------------------------------------------------------------
# Code trace support - before 105.5 - after 105.5
# ----------------------------------------------------------------------------
#if(CV_TRACE)
#  include(cmake/OpenCVDetectTrace.cmake)
#endif()

#ocv_cmake_hook(POST_DETECT_DEPENDECIES)  # typo, deprecated (2019-06)
#ocv_cmake_hook(POST_DETECT_DEPENDENCIES)

# ----------------------------------------------------------------------------
# Solution folders:
# ----------------------------------------------------------------------------
if(ENABLE_SOLUTION_FOLDERS)
  set_property(GLOBAL PROPERTY USE_FOLDERS ON)
  set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "CMakeTargets")
endif()

# Extra OpenCV targets: uninstall, package_source, perf, etc.
include(cmake/OpenCVExtraTargets.cmake)

# ----------------------------------------------------------------------------
# Process subdirectories
# ----------------------------------------------------------------------------

# opencv.hpp and legacy headers
add_subdirectory(include)

# Enable compiler options for OpenCV modules/apps/samples only (ignore 3rdparty)
#ocv_add_modules_compiler_options()

# OpenCV modules
add_subdirectory(modules)

# Generate targets for documentation
# add_subdirectory(doc)

# various data that is used by cv libraries and/or demo applications.
# add_subdirectory(data)

# extra applications
#if(BUILD_opencv_apps)
#  add_subdirectory(apps)
#endif()

# examples
# if(BUILD_EXAMPLES OR BUILD_ANDROID_EXAMPLES OR INSTALL_ANDROID_EXAMPLES OR INSTALL_PYTHON_EXAMPLES OR INSTALL_C_EXAMPLES)
#  add_subdirectory(samples)
# endif()

#if(ANDROID)
#  add_subdirectory(platforms/android/service)
# endif()

# ----------------------------------------------------------------------------
# Finalization: generate configuration-based files - before 109.5 - after 
# ----------------------------------------------------------------------------

ocv_cmake_hook(PRE_FINALIZE)

# Generate platform-dependent and configuration-dependent headers
include(cmake/OpenCVGenHeaders.cmake)

# Generate opencv.pc for pkg-config command
if(OPENCV_GENERATE_PKGCONFIG)
  include(cmake/OpenCVGenPkgconfig.cmake)
endif()

# Generate OpenCV.mk for ndk-build (Android build tool)
#include(cmake/OpenCVGenAndroidMK.cmake)

# Generate OpenCVConfig.cmake and OpenCVConfig-version.cmake for cmake projects
include(cmake/OpenCVGenConfig.cmake)

# Generate Info.plist for the IOS framework
if(APPLE_FRAMEWORK)
  include(cmake/OpenCVGenInfoPlist.cmake)
endif()

# Generate ABI descriptor
#include(cmake/OpenCVGenABI.cmake)

# Generate environment setup file
#if(INSTALL_TESTS AND OPENCV_TEST_DATA_PATH)
#  if(ANDROID)
#    get_filename_component(TEST_PATH ${OPENCV_TEST_INSTALL_PATH} DIRECTORY)
#    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/templates/opencv_run_all_tests_android.sh.in"
#                   "${CMAKE_BINARY_DIR}/unix-install/opencv_run_all_tests.sh" @ONLY)
#    install(PROGRAMS "${CMAKE_BINARY_DIR}/unix-install/opencv_run_all_tests.sh"
#            DESTINATION ./ COMPONENT tests)
#  elseif(WIN32)
#    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/templates/opencv_run_all_tests_windows.cmd.in"
#                   "${CMAKE_BINARY_DIR}/win-install/opencv_run_all_tests.cmd" @ONLY)
#    install(PROGRAMS "${CMAKE_BINARY_DIR}/win-install/opencv_run_all_tests.cmd"
#            DESTINATION ${OPENCV_TEST_INSTALL_PATH} COMPONENT tests)
#  elseif(UNIX)
#    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/templates/opencv_run_all_tests_unix.sh.in"
#                   "${CMAKE_BINARY_DIR}/unix-install/opencv_run_all_tests.sh" @ONLY)
#    install(PROGRAMS "${CMAKE_BINARY_DIR}/unix-install/opencv_run_all_tests.sh"
#            DESTINATION ${OPENCV_TEST_INSTALL_PATH} COMPONENT tests)
#  endif()
#endif()

#if(NOT OPENCV_README_FILE)
#  if(ANDROID)
#    set(OPENCV_README_FILE ${CMAKE_CURRENT_SOURCE_DIR}/platforms/android/README.android)
#  endif()
#endif()

if(NOT OPENCV_LICENSE_FILE)
  set(OPENCV_LICENSE_FILE ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)
endif()

# Tingchiun try xcode settings in here before 105.5 after 105.5
#set(CMAKE_XCODE_ATTRIBUTE_DEPLOYMENT_POSTPROCESSING "YES")
set(CMAKE_XCODE_ATTRIBUTE_DEAD_CODE_STRIPPING "YES")
set(CMAKE_XCODE_ATTRIBUTE_COPY_PHASE_STRIP "YES")
set(CMAKE_XCODE_ATTRIBUTE_STRIP_INSTALLED_PRODUCT "YES")
set(CMAKE_XCODE_ATTRIBUTE_GCC_OPTIMIZATION_LEVEL "s")

# for UNIX it does not make sense as LICENSE and readme will be part of the package automatically
if(ANDROID OR NOT UNIX)
  install(FILES ${OPENCV_LICENSE_FILE}
        PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
        DESTINATION ./ COMPONENT libs)
  if(OPENCV_README_FILE)
    install(FILES ${OPENCV_README_FILE}
            PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
            DESTINATION ./ COMPONENT libs)
  endif()
endif()

#if(COMMAND ocv_pylint_finalize)
#  ocv_pylint_add_directory(${CMAKE_CURRENT_LIST_DIR}/modules/python/test)
#  ocv_pylint_add_directory(${CMAKE_CURRENT_LIST_DIR}/samples/python)
#  ocv_pylint_add_directory(${CMAKE_CURRENT_LIST_DIR}/samples/dnn)
#  ocv_pylint_add_directory_recurse(${CMAKE_CURRENT_LIST_DIR}/samples/python/tutorial_code)
#  ocv_pylint_finalize()
#endif()

if(OPENCV_GENERATE_SETUPVARS)
  include(cmake/OpenCVGenSetupVars.cmake)
endif()

# ----------------------------------------------------------------------------
# Summary:
# ----------------------------------------------------------------------------
status("")
status("General configuration for OpenCV ${OPENCV_VERSION} =====================================")
if(OPENCV_VCSVERSION)
  status("  Version control:" ${OPENCV_VCSVERSION})
endif()
if(OPENCV_EXTRA_MODULES_PATH AND NOT BUILD_INFO_SKIP_EXTRA_MODULES)
  set(__dump_extra_header OFF)
  foreach(p ${OPENCV_EXTRA_MODULES_PATH})
    if(EXISTS ${p})
      if(NOT __dump_extra_header)
        set(__dump_extra_header ON)
        status("")
        status("  Extra modules:")
      else()
        status("")
      endif()
      ocv_git_describe(EXTRA_MODULES_VCSVERSION "${p}")
      status("    Location (extra):" ${p})
      status("    Version control (extra):" ${EXTRA_MODULES_VCSVERSION})
    endif()
  endforeach()
  unset(__dump_extra_header)
endif()

# ========================== build platform ==========================
status("")
status("  Platform:")
if(NOT DEFINED OPENCV_TIMESTAMP
    AND NOT CMAKE_VERSION VERSION_LESS 2.8.11
    AND NOT BUILD_INFO_SKIP_TIMESTAMP
)
  string(TIMESTAMP OPENCV_TIMESTAMP "" UTC)
  set(OPENCV_TIMESTAMP "${OPENCV_TIMESTAMP}" CACHE STRING "Timestamp of OpenCV build configuration" FORCE)
endif()
if(OPENCV_TIMESTAMP)
  status("    Timestamp:"      ${OPENCV_TIMESTAMP})
endif()
status("    Host:"             ${CMAKE_HOST_SYSTEM_NAME} ${CMAKE_HOST_SYSTEM_VERSION} ${CMAKE_HOST_SYSTEM_PROCESSOR})
if(CMAKE_CROSSCOMPILING)
  status("    Target:"         ${CMAKE_SYSTEM_NAME} ${CMAKE_SYSTEM_VERSION} ${CMAKE_SYSTEM_PROCESSOR})
endif()
status("    CMake:"            ${CMAKE_VERSION})
status("    CMake generator:"  ${CMAKE_GENERATOR})
status("    CMake build tool:" ${CMAKE_BUILD_TOOL})
if(MSVC)
  status("    MSVC:"           ${MSVC_VERSION})
endif()
if(CMAKE_GENERATOR MATCHES Xcode)
  status("    Xcode:"          ${XCODE_VERSION})
endif()
if(NOT CMAKE_GENERATOR MATCHES "Xcode|Visual Studio")
  status("    Configuration:"  ${CMAKE_BUILD_TYPE})
endif()


# ========================= CPU code generation mode =========================
status("")
status("  CPU/HW features:")
status("    Baseline:"  "${CPU_BASELINE_FINAL}")
if(NOT CPU_BASELINE STREQUAL CPU_BASELINE_FINAL)
  status("      requested:"  "${CPU_BASELINE}")
endif()
if(CPU_BASELINE_REQUIRE)
  status("      required:"  "${CPU_BASELINE_REQUIRE}")
endif()
if(CPU_BASELINE_DISABLE)
  status("      disabled:"  "${CPU_BASELINE_DISABLE}")
endif()
if(CPU_DISPATCH_FINAL OR CPU_DISPATCH)
  status("    Dispatched code generation:"  "${CPU_DISPATCH_FINAL}")
  if(NOT CPU_DISPATCH STREQUAL CPU_DISPATCH_FINAL)
    status("      requested:"  "${CPU_DISPATCH}")
  endif()
  if(CPU_DISPATCH_REQUIRE)
    status("      required:"  "${CPU_DISPATCH_REQUIRE}")
  endif()
  foreach(OPT ${CPU_DISPATCH_FINAL})
    status("      ${OPT} (${CPU_${OPT}_USAGE_COUNT} files):"  "+ ${CPU_DISPATCH_${OPT}_INCLUDED}")
  endforeach()
endif()

# ========================== C/C++ options ==========================
if(CMAKE_CXX_COMPILER_VERSION)
  set(OPENCV_COMPILER_STR "${CMAKE_CXX_COMPILER} ${CMAKE_CXX_COMPILER_ARG1} (ver ${CMAKE_CXX_COMPILER_VERSION})")
else()
  set(OPENCV_COMPILER_STR "${CMAKE_CXX_COMPILER} ${CMAKE_CXX_COMPILER_ARG1}")
endif()
string(STRIP "${OPENCV_COMPILER_STR}" OPENCV_COMPILER_STR)


status("")
status("  C/C++:")
status("    Built as dynamic libs?:" BUILD_SHARED_LIBS THEN YES ELSE NO)
if(DEFINED CMAKE_CXX_STANDARD AND CMAKE_CXX_STANDARD)
  status("    C++ standard:"           "${CMAKE_CXX_STANDARD}")
endif()
status("    C++ Compiler:"           ${OPENCV_COMPILER_STR})
status("    C++ flags (Release):"    ${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_RELEASE})
status("    C++ flags (Debug):"      ${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_DEBUG})
status("    C Compiler:"             ${CMAKE_C_COMPILER} ${CMAKE_C_COMPILER_ARG1})
status("    C flags (Release):"      ${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_RELEASE})
status("    C flags (Debug):"        ${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_DEBUG})
if(WIN32)
  status("    Linker flags (Release):" ${CMAKE_EXE_LINKER_FLAGS} ${CMAKE_EXE_LINKER_FLAGS_RELEASE})
  status("    Linker flags (Debug):"   ${CMAKE_EXE_LINKER_FLAGS} ${CMAKE_EXE_LINKER_FLAGS_DEBUG})
else()
  status("    Linker flags (Release):" ${CMAKE_SHARED_LINKER_FLAGS} ${CMAKE_SHARED_LINKER_FLAGS_RELEASE})
  status("    Linker flags (Debug):"   ${CMAKE_SHARED_LINKER_FLAGS} ${CMAKE_SHARED_LINKER_FLAGS_DEBUG})
endif()
status("    ccache:"                  OPENCV_COMPILER_IS_CCACHE THEN YES ELSE NO)
status("    Precompiled headers:"     PCHSupport_FOUND AND ENABLE_PRECOMPILED_HEADERS THEN YES ELSE NO)

# ========================== Dependencies ============================
# ocv_get_all_libs(deps_modules deps_extra deps_3rdparty)
# status("    Extra dependencies:" ${deps_extra})
# status("    3rdparty dependencies:" ${deps_3rdparty})

# ========================== OpenCV modules ==========================
status("")
status("  OpenCV modules:")
set(OPENCV_MODULES_BUILD_ST "")
foreach(the_module ${OPENCV_MODULES_BUILD})
  if(NOT OPENCV_MODULE_${the_module}_CLASS STREQUAL "INTERNAL" OR the_module STREQUAL "opencv_ts")
    list(APPEND OPENCV_MODULES_BUILD_ST "${the_module}")
  endif()
endforeach()
string(REPLACE "opencv_" "" OPENCV_MODULES_BUILD_ST          "${OPENCV_MODULES_BUILD_ST}")
string(REPLACE "opencv_" "" OPENCV_MODULES_DISABLED_USER_ST  "${OPENCV_MODULES_DISABLED_USER}")
string(REPLACE "opencv_" "" OPENCV_MODULES_DISABLED_AUTO_ST  "${OPENCV_MODULES_DISABLED_AUTO}")
string(REPLACE "opencv_" "" OPENCV_MODULES_DISABLED_FORCE_ST "${OPENCV_MODULES_DISABLED_FORCE}")
list(SORT OPENCV_MODULES_BUILD_ST)
list(SORT OPENCV_MODULES_DISABLED_USER_ST)
list(SORT OPENCV_MODULES_DISABLED_AUTO_ST)
list(SORT OPENCV_MODULES_DISABLED_FORCE_ST)
status("    To be built:"            OPENCV_MODULES_BUILD          THEN ${OPENCV_MODULES_BUILD_ST}          ELSE "-")
status("    Disabled:"               OPENCV_MODULES_DISABLED_USER  THEN ${OPENCV_MODULES_DISABLED_USER_ST}  ELSE "-")
status("    Disabled by dependency:" OPENCV_MODULES_DISABLED_AUTO  THEN ${OPENCV_MODULES_DISABLED_AUTO_ST}  ELSE "-")
status("    Unavailable:"            OPENCV_MODULES_DISABLED_FORCE THEN ${OPENCV_MODULES_DISABLED_FORCE_ST} ELSE "-")

ocv_build_features_string(apps_status
  IF BUILD_TESTS AND HAVE_opencv_ts THEN "tests"
  IF BUILD_PERF_TESTS AND HAVE_opencv_ts THEN "perf_tests"
  IF BUILD_EXAMPLES THEN "examples"
  IF BUILD_opencv_apps THEN "apps"
  IF BUILD_ANDROID_SERVICE THEN "android_service"
  IF (BUILD_ANDROID_EXAMPLES OR INSTALL_ANDROID_EXAMPLES) AND CAN_BUILD_ANDROID_PROJECTS THEN "android_examples"
  ELSE "-")
status("    Applications:" "${apps_status}")
ocv_build_features_string(docs_status
    IF TARGET doxygen_cpp THEN "doxygen"
    IF TARGET doxygen_python THEN "python"
    IF TARGET doxygen_javadoc THEN "javadoc"
    IF BUILD_opencv_js OR DEFINED OPENCV_JS_LOCATION THEN "js"
    ELSE "NO"
)
status("    Documentation:" "${docs_status}")
status("    Non-free algorithms:" OPENCV_ENABLE_NONFREE THEN "YES" ELSE "NO")

# ========================== Android details ==========================

# ================== Windows RT features ==================

# ========================== GUI ==========================

# ========================== MEDIA IO ==========================


# ========================== VIDEO IO ==========================


# ========================== Other third-party libraries ==========================
#status("")
#status("  Other third-party libraries:")

#if(WITH_IPP AND HAVE_IPP)
#  status("    Intel IPP:" "${IPP_VERSION_STR} [${IPP_VERSION_MAJOR}.${IPP_VERSION_MINOR}.${IPP_VERSION_BUILD}]")
#  status("           at:" "${IPP_ROOT_DIR}")
#  if(NOT HAVE_IPP_ICV)
#    status("       linked:" BUILD_WITH_DYNAMIC_IPP THEN "dynamic" ELSE "static")
#  endif()
#  if(HAVE_IPP_IW)
#    if(BUILD_IPP_IW)
#      status("    Intel IPP IW:" "sources (${IW_VERSION_MAJOR}.${IW_VERSION_MINOR}.${IW_VERSION_UPDATE})")
#    else()
#      status("    Intel IPP IW:" "binaries (${IW_VERSION_MAJOR}.${IW_VERSION_MINOR}.${IW_VERSION_UPDATE})")
#    endif()
#    status("              at:" "${IPP_IW_PATH}")
#  else()
#    status("    Intel IPP IW:"   NO)
#  endif()
#endif()

#if(WITH_VA OR HAVE_VA)
#  status("    VA:"            HAVE_VA          THEN "YES" ELSE NO)
#endif()

#if(WITH_TENGINE OR HAVE_TENGINE)
#  status("    Tengine:"      HAVE_TENGINE     THEN "YES (${TENGINE_LIBRARIES})" ELSE NO)
#endif()

#if(WITH_LAPACK OR HAVE_LAPACK)
#  status("    Lapack:"      HAVE_LAPACK     THEN "YES (${LAPACK_LIBRARIES})" ELSE NO)
#endif()

#if(WITH_HALIDE OR HAVE_HALIDE)
#  status("    Halide:"     HAVE_HALIDE      THEN "YES (${HALIDE_LIBRARIES} ${HALIDE_INCLUDE_DIRS})" ELSE NO)
#endif()

#if(WITH_INF_ENGINE OR INF_ENGINE_TARGET)
#  if(INF_ENGINE_TARGET)
#    list(GET INF_ENGINE_TARGET 0 ie_target)
#    set(__msg "YES (${INF_ENGINE_RELEASE} / ${INF_ENGINE_VERSION})")
#    get_target_property(_lib ${ie_target} IMPORTED_LOCATION)
#    get_target_property(_lib_imp_rel ${ie_target} IMPORTED_IMPLIB_RELEASE)
#    get_target_property(_lib_imp_dbg ${ie_target} IMPORTED_IMPLIB_DEBUG)
#    get_target_property(_lib_rel ${ie_target} IMPORTED_LOCATION_RELEASE)
#    get_target_property(_lib_dbg ${ie_target} IMPORTED_LOCATION_DEBUG)
#    ocv_build_features_string(_lib
#     IF _lib THEN "${_lib}"
#      IF _lib_imp_rel AND _lib_imp_dbg THEN "${_lib_imp_rel} / ${_lib_imp_dbg}"
#      IF _lib_rel AND _lib_dbg THEN "${_lib_rel} / ${_lib_dbg}"
#      IF _lib_rel  THEN "${_lib_rel}"
#      IF _lib_dbg  THEN "${_lib_dbg}"
#      ELSE "unknown"
#    )
#    get_target_property(_inc ${ie_target} INTERFACE_INCLUDE_DIRECTORIES)
#    status("    Inference Engine:" "${__msg}")
#    status("        * libs:" "${_lib}")
#    status("        * includes:" "${_inc}")
#  else()
#    status("    Inference Engine:"     "NO")
#  endif()
#endif()
#if(WITH_NGRAPH OR HAVE_NGRAPH)
#  if(HAVE_NGRAPH)
#    set(__target ngraph::ngraph)
#    set(__msg "YES (${ngraph_VERSION})")
#    get_target_property(_lib ${__target} IMPORTED_LOCATION)
#    get_target_property(_lib_imp_rel ${__target} IMPORTED_IMPLIB_RELEASE)
#    get_target_property(_lib_imp_dbg ${__target} IMPORTED_IMPLIB_DEBUG)
#    get_target_property(_lib_rel ${__target} IMPORTED_LOCATION_RELEASE)
#    get_target_property(_lib_dbg ${__target} IMPORTED_LOCATION_DEBUG)
#    ocv_build_features_string(_lib
#      IF _lib THEN "${_lib}"
#      IF _lib_imp_rel AND _lib_imp_dbg THEN "${_lib_imp_rel} / ${_lib_imp_dbg}"
#      IF _lib_rel AND _lib_dbg THEN "${_lib_rel} / ${_lib_dbg}"
#      IF _lib_rel  THEN "${_lib_rel}"
#      IF _lib_dbg  THEN "${_lib_dbg}"
#      ELSE "unknown"
#    )
#    get_target_property(_inc ${__target} INTERFACE_INCLUDE_DIRECTORIES)
#    status("    nGraph:" "${__msg}")
#    status("        * libs:" "${_lib}")
#    status("        * includes:" "${_inc}")
#  else()
#    status("    nGraph:"     "NO")
#  endif()
#endif()

#if(WITH_EIGEN OR HAVE_EIGEN)
#  status("    Eigen:"      HAVE_EIGEN       THEN "YES (ver ${EIGEN_WORLD_VERSION}.${EIGEN_MAJOR_VERSION}.${EIGEN_MINOR_VERSION})" ELSE NO)
#endif()

#if(WITH_OPENVX OR HAVE_OPENVX)
#  status("    OpenVX:"     HAVE_OPENVX      THEN "YES (${OPENVX_LIBRARIES})" ELSE "NO")
#endif()

#status("    Custom HAL:" OpenCV_USED_HAL  THEN "YES (${OpenCV_USED_HAL})" ELSE "NO")

#foreach(s ${CUSTOM_STATUS})
#  status(${CUSTOM_STATUS_${s}})
#endforeach()

#if(WITH_CUDA OR HAVE_CUDA)
#  ocv_build_features_string(cuda_features
#    IF HAVE_CUFFT THEN "CUFFT"
#    IF HAVE_CUBLAS THEN "CUBLAS"
#    IF HAVE_NVCUVID THEN "NVCUVID"
#    IF CUDA_FAST_MATH THEN "FAST_MATH"
#    ELSE "no extra features")
#  status("")
#  status("  NVIDIA CUDA:" HAVE_CUDA THEN "YES (ver ${CUDA_VERSION_STRING}, ${cuda_features})" ELSE NO)
#  if(HAVE_CUDA)
#    status("    NVIDIA GPU arch:"      ${OPENCV_CUDA_ARCH_BIN})
#    status("    NVIDIA PTX archs:"     ${OPENCV_CUDA_ARCH_PTX})
#  endif()
# endif()

# if(WITH_CUDNN OR HAVE_CUDNN)
#    status("")
#    status("  cuDNN:" HAVE_CUDNN THEN "YES (ver ${CUDNN_VERSION})" ELSE NO)
#endif()

#if(WITH_VULKAN OR HAVE_VULKAN)
#  status("")
#  status("  Vulkan:"     HAVE_VULKAN THEN "YES" ELSE "NO")
#  if(HAVE_VULKAN)
#    status("    Include path:"  VULKAN_INCLUDE_DIRS THEN "${VULKAN_INCLUDE_DIRS}" ELSE "NO")
#    status("    Link libraries:" VULKAN_LIBRARIES THEN "${VULKAN_LIBRARIES}" ELSE "Dynamic load")
#  endif()
#endif()

#if(WITH_OPENCL OR HAVE_OPENCL)
#  ocv_build_features_string(opencl_features
#    IF HAVE_OPENCL_SVM THEN "SVM"
#    IF HAVE_CLAMDFFT THEN "AMDFFT"
#    IF HAVE_CLAMDBLAS THEN "AMDBLAS"
#    IF HAVE_OPENCL_D3D11_NV THEN "NVD3D11"
#    IF HAVE_VA_INTEL THEN "INTELVA"
#    ELSE "no extra features")
#  status("")
#  status("  OpenCL:"     HAVE_OPENCL   THEN   "YES (${opencl_features})" ELSE "NO")
#  if(HAVE_OPENCL)
#    status("    Include path:"  OPENCL_INCLUDE_DIRS THEN "${OPENCL_INCLUDE_DIRS}" ELSE "NO")
#    status("    Link libraries:"       OPENCL_LIBRARIES THEN "${OPENCL_LIBRARIES}" ELSE "Dynamic load")
#  endif()
#endif()

# ========================== python ==========================

# ========================== java ==========================


# ========================== Objective-C =======================
if(BUILD_OBJC)
  status("")
  status("  Objective-C wrappers:" HAVE_opencv_objc                                                       THEN YES ELSE NO)
endif()

ocv_cmake_hook(STATUS_DUMP_EXTRA)

# ========================== auxiliary ==========================
status("")
status("  Install to:" "${CMAKE_INSTALL_PREFIX}")
status("-----------------------------------------------------------------")
status("")


ocv_finalize_status()

if(ENABLE_CONFIG_VERIFICATION)
  ocv_verify_config()
endif()

if(HAVE_CUDA AND COMMAND CUDA_BUILD_CLEAN_TARGET)
  CUDA_BUILD_CLEAN_TARGET()
endif()

ocv_cmake_hook(POST_FINALIZE)

# ----------------------------------------------------------------------------
# CPack stuff
# ----------------------------------------------------------------------------

include(cmake/OpenCVPackaging.cmake)

# This should be the last command
ocv_cmake_dump_vars("" TOFILE "CMakeVars.txt")
ocv_cmake_eval(DEBUG_POST ONCE)
