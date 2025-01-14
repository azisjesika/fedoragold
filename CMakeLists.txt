cmake_minimum_required(VERSION 2.8.6)

set(VERSION "0.2")
# $Format:Packaged from commit %H%nset(COMMIT %h)%nset(REFS "%d")$

#set(CMAKE_EXE_LINK_DYNAMIC_C_FLAGS)       # remove -Wl,-Bdynamic
#set(CMAKE_EXE_LINK_DYNAMIC_CXX_FLAGS)
#set(CMAKE_SHARED_LIBRARY_C_FLAGS)         # remove -fPIC
#set(CMAKE_SHARED_LIBRARY_CXX_FLAGS)
#set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)    # remove -rdynamic
#set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)

# The following are: -triple" "i686-pc-windows-msvc19.12.25831" "-emit-obj" "-mrelax-all" 
# "-mincremental-linker-compatible" "-disable-free" "-main-file-name" "open.c" 
# "-mrelocation-model" "static" "-mthread-model" "posix" "-mdisable-fp-elim" 
# "-fmath-errno" "-masm-verbose" "-mconstructor-aliases" "-target-cpu" "pentium4"
# ...etc. I have specified from where to get the libraries with -isystem "path_to_lib". 
# The command was something like clang -cc1 -isystem "path" --analyze...
#
# -flto -fno-autolink -save-temps 
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -flto=thin")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto=thin")

SET(CMAKE_RANLIB "/usr/bin/llvm-ranlib" CACHE INTERNAL STRING)
SET(CMAKE_AR "/usr/bin/llvm-ar" CACHE INTERNAL STRING)
#SET(CMAKE_LINKER "/usr/bin/llvm-ld" CACHE INTERNAL STRING)

enable_language(CXX)
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads REQUIRED)

set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_MULTITHREADED ON)
set(Boost_USE_STATIC_RUNTIME ON)
set(CMAKE_FIND_LIBRARY_SUFFIXES .a ${CMAKE_FIND_LIBRARY_SUFFIXES})

# use this to generate documentation for our REST api's
# set(LLVM_ENABLE_SPHINX "ON")

set(CMAKE_SYSTEM_NAME "Linux")
set(CMAKE_CROSSCOMPILING "ON")
set(LLVM_DEFAULT_TARGET_TRIPLE "x86_64-pc-linux-musl")
set(LLVM_TARGETS_TO_BUILD "X86")
set(LLVM_TARGET_ARCH "x86_64")

#set(LLVM_ENABLE_ASSERTIONS "ON")
#set(LLVM_ENABLE_THREADS "ON") 
#set(LLVM_ENABLE_LTO "ON")
#set(LLVM_OPTIMIZED_TABLEGEN "ON")
#set(LLVM_USE_OPROFILE "ON")
# LLVM_CREATE_XCODE_TOOLCHAIN:
# LLVM_BUILD_DOCS
# LLVM_ENABLE_DOXYGEN

if(APPLE)
#increase default OSX stack size to 256 MB
#set(CMAKE_EXE_LINKER_FLAGS "-Wl,-stack_size,0x10000000,-stack_addr,0xc0000000")
else()
# -static-libgcc -static-libstdc++ 
set(CMAKE_EXE_LINKER_FLAGS "-fuse-ld=gold -use-gold-plugin -static -pthread -Wl,-Bstatic -Llibc.a -Wl,-Bstatic -Llibm.a")
endif()
set(BOOST_DEBUG ON) #joja
set(BOOST_HAS_THREADS ON)
set(BOOST_HAS_PTHREADS ON) 
set(BUILD_SHARED_LIBS OFF)
set(STATIC ON CACHE BOOL "Link libraries statically")
set(BOOST_THREAD_LIBRARY libboost_thread.a)
add_library(c STATIC IMPORTED)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED on)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_CONFIGURATION_TYPES "Debug RelWithDebInfo Release CACHE TYPE INTERNAL")

#joja what is this stuff?
set(CMAKE_SKIP_INSTALL_RULES ON)
set(CMAKE_SKIP_PACKAGE_ALL_DEPENDENCY ON)
set(CMAKE_SUPPRESS_REGENERATION ON)

#joja what is this?
enable_testing()

# copy CTestCustom.cmake to build dir to disable long running tests in 'make test'
configure_file(${CMAKE_SOURCE_DIR}/CTestCustom.cmake ${CMAKE_BINARY_DIR})

project(CryptoNote)

include_directories(include src external "${CMAKE_BINARY_DIR}/version")

enable_language(ASM)

if(APPLE)
  include_directories(SYSTEM /usr/include/malloc)
#  enable_language(ASM)
endif()

if(MSVC)
include_directories(src/Platform/Windows)
elseif(APPLE)
include_directories(src/Platform/OSX)
else()
include_directories(src/Platform/Linux)
include_directories(SYSTEM /usr/include)
endif()

# set(STATIC ${MSVC} CACHE BOOL "Link libraries statically")

if(MSVC)
  add_definitions("/bigobj /MP /W3 /GS- /D_CRT_SECURE_NO_WARNINGS /wd4996 /wd4345 /D_WIN32_WINNT=0x0600 /DWIN32_LEAN_AND_MEAN /DGTEST_HAS_TR1_TUPLE=0 /D_VARIADIC_MAX=8 /D__SSE4_1__")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /STACK:10485760")
  if(STATIC)
    foreach(VAR CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_DEBUG CMAKE_C_FLAGS_RELWITHDEBINFO CMAKE_CXX_FLAGS_RELWITHDEBINFO CMAKE_C_FLAGS_RELEASE CMAKE_CXX_FLAGS_RELEASE)
      string(REPLACE "/MD" "/MT" ${VAR} "${${VAR}}")
    endforeach()
  endif()
  include_directories(SYSTEM src/platform/msc)
else()
  if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    # This option has no effect in glibc version less than 2.20. 
    # Since glibc 2.20 _BSD_SOURCE is deprecated, this macro is recomended instead
    add_definitions("-D_DEFAULT_SOURCE -D_GNU_SOURCE")
  endif()
  set(ARCH native CACHE STRING "CPU to build for: -march value or default")
  if("${ARCH}" STREQUAL "default")
    set(ARCH_FLAG "")
  else()
    set(ARCH_FLAG "-march=${ARCH}")
  endif()
  set(WARNINGS "-Wall -Wextra -Wpointer-arith -Wundef -Wvla -Wwrite-strings -Werror -Wno-error=extra -Wno-error=unused-function -Wno-error=deprecated-declarations -Wno-error=sign-compare -Wno-error=strict-aliasing -Wno-error=type-limits -Wno-unused-parameter -Wno-error=unused-variable -Wno-error=undef -Wno-error=uninitialized -Wno-error=unused-result")
  if(CMAKE_C_COMPILER_ID STREQUAL "Clang")
    set(WARNINGS "${WARNINGS} -Wno-error=mismatched-tags -Wno-error=null-conversion -Wno-overloaded-shift-op-parentheses -Wno-error=shift-count-overflow -Wno-error=tautological-constant-out-of-range-compare -Wno-error=unused-private-field -Wno-error=unneeded-internal-declaration -Wno-error=unused-function")
  else()
    set(WARNINGS "${WARNINGS} -Wlogical-op -Wno-error=maybe-uninitialized -Wno-error=clobbered -Wno-error=unused-but-set-variable")
  endif()
  if(MINGW)
    set(WARNINGS "${WARNINGS} -Wno-error=unused-value")
    set(MINGW_FLAG "-DWIN32_LEAN_AND_MEAN")
    include_directories(SYSTEM src/platform/mingw)
  else()
    set(MINGW_FLAG "")
  endif()
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 5.1))
    set(WARNINGS "${WARNINGS} -Wno-error=odr")
  endif()
  set(C_WARNINGS "-Waggregate-return -Wnested-externs -Wold-style-definition -Wstrict-prototypes")
  set(CXX_WARNINGS "-Wno-reorder -Wno-missing-field-initializers")

  #set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${MINGW_FLAG} ${WARNINGS} ${C_WARNINGS} ${ARCH_FLAG} -maes")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 ${MINGW_FLAG} ${WARNINGS} ${C_WARNINGS} ${ARCH_FLAG} -maes") 
  if(NOT APPLE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static -pthread")

  endif()
  #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${MINGW_FLAG} ${WARNINGS} ${CXX_WARNINGS} ${ARCH_FLAG} -maes")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 ${MINGW_FLAG} ${WARNINGS} ${CXX_WARNINGS} ${ARCH_FLAG} -maes")
  if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGTEST_HAS_TR1_TUPLE=0")
  endif()
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 4.8))
    set(DEBUG_FLAGS "-g3 -Og")
  else()
    set(DEBUG_FLAGS "-g3 -O0")
  endif()
  set(RELEASE_FLAGS "-Ofast -DNDEBUG -Wno-unused-variable")
  if(NOT APPLE)
    # There is a clang bug that does not allow to compile code that uses AES-NI intrinsics if -flto is enabled
#    if (CMAKE_C_COMPILER_ID STREQUAL "GNU" AND CMAKE_SYSTEM_NAME STREQUAL "Linux" 
#        AND CMAKE_BUILD_TYPE STREQUAL "Release" AND ((CMAKE_C_COMPILER_VERSION VERSION_GREATER 4.9) OR (CMAKE_C_COMPILER_VERSION VERSION_EQUAL 4.9)))
#      # On linux, to build in lto mode, check that ld.gold linker is used: 'update-alternatives --install /usr/bin/ld ld /usr/bin/ld.gold HIGHEST_PRIORITY'
#jojapoppa
#      set(CMAKE_AR gcc-ar)
#      set(CMAKE_RANLIB gcc-ranlib)
#    endif()
    set(RELEASE_FLAGS "${RELEASE_FLAGS} -flto")
  endif()
  #if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT MINGW)
  #  set(RELEASE_FLAGS "${RELEASE_FLAGS} -fno-fat-lto-objects")
  #endif()
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${RELEASE_FLAGS}")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${RELEASE_FLAGS}")

  #joja
  #if(STATIC AND NOT APPLE)
  #  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++11") #joja 11
  #endif()

endif()

if(STATIC)
  set(Boost_USE_STATIC_LIBS ON)
  set(Boost_USE_STATIC_RUNTIME ON)
endif()
find_package(Boost 1.55 REQUIRED COMPONENTS system filesystem thread date_time chrono regex serialization program_options)
include_directories(SYSTEM ${Boost_INCLUDE_DIRS})
if(MINGW)
  set(Boost_LIBRARIES "${Boost_LIBRARIES};ws2_32;mswsock")
elseif(APPLE)
  set(Boost_LIBRARIES "${Boost_LIBRARIES}")
elseif(NOT MSVC)
  set(Boost_LIBRARIES "${Boost_LIBRARIES};rt")
endif()

set(COMMIT_ID_IN_VERSION ON CACHE BOOL "Include commit ID in version")
file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/version")
if (NOT COMMIT_ID_IN_VERSION)
  set(VERSION "${VERSION}-unknown")
  configure_file("src/version.h.in" "version/version.h")
  add_custom_target(version ALL)
elseif(DEFINED COMMIT)
  string(REPLACE "." "\\." VERSION_RE "${VERSION}")
  if(NOT REFS MATCHES "(\\(|, )tag: v${VERSION_RE}(\\)|, )")
    set(VERSION "${VERSION}-g${COMMIT}")
  endif()
  configure_file("src/version.h.in" "version/version.h")
  add_custom_target(version ALL)
else()
  find_package(Git QUIET)
  if(Git_FOUND OR GIT_FOUND)
    message(STATUS "Found Git: ${GIT_EXECUTABLE}")
    add_custom_target(version ALL "${CMAKE_COMMAND}" "-D" "VERSION=${VERSION}" "-D" "GIT=${GIT_EXECUTABLE}" "-D" "TO=${CMAKE_BINARY_DIR}/version/version.h" "-P" "src/version.cmake" WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}")
  else()
    message(STATUS "WARNING: Git was not found!")
    set(VERSION "${VERSION}-unknown")
    configure_file("src/version.h.in" "version/version.h")
    add_custom_target(version ALL)
  endif()
endif()

add_subdirectory(external)
add_subdirectory(src)
add_subdirectory(tests)
