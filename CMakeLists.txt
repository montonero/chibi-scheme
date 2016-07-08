cmake_minimum_required(VERSION 3.5)
project(chibi-scheme)

# detect the platform
set(PLATFORM_DETECT linux)
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
  set(PLATFORM_DETECT macosx)
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
  set(PLATFORM_DETECT bsd)
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD")
  set(PLATFORM_DETECT bsd)
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
  set(PLATFORM_DETECT bsd)
endif()
if(${MINGW})
  set(PLATFORM_DETECT mingw)
endif()
if(${CYGWIN})
  set(PLATFORM_DETECT cygwin)
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
  set(PLATFORM_DETECT linux)
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "Emscripten")
  set(PLATFORM_DETECT emscripten)
endif()
if(${MSVC})
  set(PLATFORM_DETECT msvc)
endif()

set(CHIBI_CROSS_BINARY_DIR "" CACHE STRING "")
set(CHIBI_CROSS_PLATFORM "" CACHE STRING "")
set(CHIBI_PLATFORM ${PLATFORM_DETECT} CACHE STRING "")

set(OPTIMIZATION_FLAGS "-O2" CACHE STRING "")

option(BUILD_SHARED_LIBS "BUILD_SHARED_LIBS" ON)

if(${CHIBI_PLATFORM} MATCHES "msvc")
  set(OPTIMIZATION_FLAGS "")
  set(SEXP_USE_GREEN_THREADS OFF)
  set(SEXP_USE_IMAGE_LOADING OFF)
endif()

# build static libs when compiling with emscripten
if(${CHIBI_PLATFORM} MATCHES "emscripten")
  set(SEXP_USE_STATIC_LIBS ON)
  set(SEXP_USE_DL OFF)
  set(SEXP_USE_STRICT_TOP_LEVEL_BINDINGS ON)
  set(SEXP_USE_ALIGNED_BYTECODE ON)
endif()

include(CheckFunctionExists)
# ntp_gettime doesn't work for static libs?
set(SEXP_USE_NTP_GETTIME 0)
if(NOT ${SEXP_USE_STATIC_LIBS})
  set(CMAKE_EXTRA_INCLUDE_FILES sys/timex.h)
  check_function_exists(ntp_gettime SEXP_USE_NTP_GETTIME)
endif()

include(CheckTypeSize)
check_type_size(int_least8_t SEXP_USE_INTTYPES)

set(CHIBI_EXECUTABLE $<TARGET_FILE:chibi-scheme>)

# static libs requires a cross compiler
if(${SEXP_USE_STATIC_LIBS})
  include(${CHIBI_CROSS_BINARY_DIR}/exports.cmake)
  set(CHIBI_EXECUTABLE $<TARGET_FILE:external-chibi-scheme>)
endif()

# Generate the install.h file
set(MODDIR ${CMAKE_INSTALL_PREFIX}/share/chibi)
set(BINMODDIR ${CMAKE_INSTALL_PREFIX}/lib/chibi)

set(PLATFORM ${CHIBI_PLATFORM})
if(NOT ${CHIBI_CROSS_PLATFORM} STREQUAL "")
  set(PLATFORM ${CHIBI_CROSS_PLATFORM})
endif()

file(STRINGS VERSION VERSION)
file(STRINGS RELEASE RELEASE)
file(WRITE ${PROJECT_BINARY_DIR}/include/chibi/install.h
  "
#define sexp_so_extension \"${CMAKE_SHARED_LIBRARY_SUFFIX}\"
#define sexp_default_module_path \"${MODDIR}:${BINMODDIR}\"
#define sexp_platform \"${PLATFORM}\"
#define sexp_version \"${VERSION}\"
#define sexp_release_name \"${RELEASE}\"
")

# copy lib, include, and opt into build dir
file(COPY ${PROJECT_SOURCE_DIR}/lib/ DESTINATION ${PROJECT_BINARY_DIR}/lib/)
file(COPY ${PROJECT_SOURCE_DIR}/include/ DESTINATION ${PROJECT_BINARY_DIR}/include/)
file(COPY ${PROJECT_SOURCE_DIR}/opt/ DESTINATION ${PROJECT_BINARY_DIR}/opt/)

file(GLOB_RECURSE SCM_FILES RELATIVE ${PROJECT_BINARY_DIR} ${PROJECT_BINARY_DIR}/lib/*.scm)
file(GLOB_RECURSE SLD_FILES RELATIVE ${PROJECT_BINARY_DIR} ${PROJECT_BINARY_DIR}/lib/*.sld)

# copy lib into runtime dirs
foreach(config_type ${CMAKE_CONFIGURATION_TYPES})
  foreach(file ${SCM_FILES})
    get_filename_component(path ${file} DIRECTORY)
    file(COPY ${file} DESTINATION ${PROJECT_BINARY_DIR}/${config_type}/${path})
  endforeach()
  foreach(file ${SLD_FILES})
    get_filename_component(path ${file} DIRECTORY)
    file(COPY ${file} DESTINATION ${PROJECT_BINARY_DIR}/${config_type}/${path})
  endforeach()
endforeach()

include_directories(${PROJECT_BINARY_DIR}/include)

macro(chibi_feature name default)
  if(NOT DEFINED SEXP_USE_${name})
    option(${name} "" ${default})
    option(SEXP_USE_${name} "" ${${name}})
  endif()
  if(NOT "${SEXP_USE_${name}}" STREQUAL "${default}")
    # show some output to indicate non default
    message(STATUS "SEXP_USE_${name}=${SEXP_USE_${name}}")
    add_definitions(-DSEXP_USE_${name}=$<BOOL:${SEXP_USE_${name}}>)
  endif()
endmacro(chibi_feature)

chibi_feature(NO_FEATURES OFF)
chibi_feature(GREEN_THREADS ON)
chibi_feature(NATIVE_X86 OFF)
chibi_feature(MODULES ON)
chibi_feature(DL ON)
chibi_feature(STATIC_LIBS OFF)
chibi_feature(FULL_SOURCE_INFO ON)
chibi_feature(SIMPLIFY ON)
chibi_feature(TYPE_DEFS ON)
chibi_feature(BOEHM OFF)
chibi_feature(WEAK ON)
chibi_feature(FIXED_CHUNK_SIZED_HEAPS OFF)
chibi_feature(MALLOC OFF)
chibi_feature(MMAP_GC OFF)
chibi_feature(CONSERVATIVE_GC OFF)
chibi_feature(FINALIZERS ON)
chibi_feature(GC_MARK OFF)
chibi_feature(TRACK_ALLOC_SOURCE OFF)
chibi_feature(HEADER_MAGIC OFF)
chibi_feature(DEBUG_GC OFF)
chibi_feature(TIME_GC OFF)
chibi_feature(SAFE_ACCESSORS OFF)
chibi_feature(PRINT_BACKTRACE_ON_SEGFAULT OFF)
chibi_feature(GLOBAL_HEAP OFF)
chibi_feature(GLOBAL_SYMBOLS OFF)
chibi_feature(EXTENDED_FCALL ON)
chibi_feature(FLONUMS ON)
chibi_feature(INFINITIES ON)
chibi_feature(IMMEDIATE_FLONUMS OFF)
chibi_feature(BIGNUMS ON)
chibi_feature(RATIOS ON)
chibi_feature(COMPLEX ON)
chibi_feature(PLACEHOLDER_DIGITS ON)
chibi_feature(MATH ON)
chibi_feature(WARN_UNDEFS ON)
chibi_feature(HUFF_SYMS ON)
chibi_feature(EXTENDED_CHAR_NAMES ON)
chibi_feature(READER_LABELS ON)
chibi_feature(UTF8-STRINGS ON)
chibi_feature(MUTABLE_STRINGS ON)
chibi_feature(DISJOINT_STRING_CURSORS ON)
chibi_feature(AUTOCLOSE_PORTS ON)
chibi_feature(2010_EPOCH OFF)
chibi_feature(CHECK_STACK ON)
chibi_feature(GROW_STACK ON)
# this isnt in features.h
chibi_feature(IMAGE_LOADING ON)
chibi_feature(STRICT_TOP_LEVEL_BINDINGS OFF)
chibi_feature(ALIGNED_BYTECODE OFF)

add_definitions(
  ${OPTIMIZATION_FLAGS}
  -DSEXP_USE_NTP_GETTIME=${SEXP_USE_NTP_GETTIME}
  -DSEXP_USE_INTTYPES=${SEXP_USE_INTTYPES})

string(REPLACE ";" "\n" SLD_FILES_GENSTATIC "${SLD_FILES}")
file(WRITE ${PROJECT_BINARY_DIR}/genstatic-input
  "${SLD_FILES_GENSTATIC}")

add_library(chibi-scheme-lib
  gc.c
  sexp.c
  bignum.c
  gc_heap.c
  opcodes.c
  vm.c
  eval.c
  simplify.c)

string(REPLACE "." ";" VERSION_LIST ${VERSION})
list(GET VERSION_LIST 0 API_VERSION)

set_target_properties(chibi-scheme-lib PROPERTIES
  OUTPUT_NAME chibi-scheme
  COMPILE_DEFINITIONS "BUILDING_DLL"
  SOVERSION ${API_VERSION}
  VERSION ${VERSION})

target_link_libraries(chibi-scheme-lib
  ${OPTIMIZATION_FLAGS}
  $<$<BOOL:${SEXP_USE_DL}>:${CMAKE_DL_LIBS}>
  $<$<BOOL:${SEXP_USE_BOEHM}>:gc>
  $<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:-static>
  $<$<NOT:$<STREQUAL:msvc,${CHIBI_PLATFORM}>>:m>
  $<$<STREQUAL:msvc,${CHIBI_PLATFORM}>:shlwapi>)

add_executable(chibi-scheme main.c)
target_link_libraries(chibi-scheme
  chibi-scheme-lib
  ${OPTIMIZATION_FLAGS}
  $<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:-static>)

if(${CHIBI_PLATFORM} MATCHES emscripten)
  foreach(file ${SCM_FILES})
    target_link_libraries(chibi-scheme
      "--preload-file ${file}")
  endforeach()
  foreach(file ${SLD_FILES})
    target_link_libraries(chibi-scheme
      "--preload-file ${file}")
  endforeach()
  target_link_libraries(chibi-scheme
    "-s MODULARIZE=1"
    "-s EXPORT_NAME=\"'Chibi'\""
    "-s EXPORTED_FUNCTIONS=\"['_main','_sexp_resume']\""
    "--pre-js ${PROJECT_SOURCE_DIR}/js/pre.js"
    "--post-js ${PROJECT_SOURCE_DIR}/js/post.js")
  set_target_properties(chibi-scheme PROPERTIES
    OUTPUT_NAME chibi)
endif()

export(TARGETS chibi-scheme
  NAMESPACE external-
  FILE ${PROJECT_BINARY_DIR}/exports.cmake)

set(CMAKE_MSVCIDE_RUN_PATH ${PROJECT_BINARY_DIR}/$<CONFIG>)

macro(chibi_module name source output_path output_name)
  option(USE_MODULE_${name} "" ON)
  if(${USE_MODULE_${name}} AND ${SEXP_USE_DL})
    add_library(${name} ${PROJECT_BINARY_DIR}/${source})
    set_target_properties(${name} PROPERTIES
      RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/$<CONFIG>/${output_path}
      LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${output_path}
      PREFIX ""
      OUTPUT_NAME ${output_name}
      COMPILE_DEFINITIONS "BUILDING_DLL")
    target_link_libraries(${name} chibi-scheme-lib)
  endif()
endmacro()

macro(chibi_ffi stub out)
  add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/${out}
    COMMAND ${CHIBI_EXECUTABLE} -q
    ${PROJECT_SOURCE_DIR}/tools/chibi-ffi
    ${PROJECT_BINARY_DIR}/${stub}
    ${PROJECT_BINARY_DIR}/${out}
    MAIN_DEPENDENCY ${stub})
  set_source_files_properties(${PROJECT_BINARY_DIR}/${out}
    PROPERTIES GENERATED TRUE)
  list(APPEND GENERATED_SOURCES ${PROJECT_BINARY_DIR}/${out})
endmacro()

chibi_ffi(lib/chibi/crypto/crypto.stub lib/chibi/crypto/crypto.c)
chibi_ffi(lib/chibi/process.stub lib/chibi/process.c)
chibi_ffi(lib/chibi/system.stub lib/chibi/system.c)
chibi_ffi(lib/chibi/time.stub lib/chibi/time.c)
chibi_ffi(lib/chibi/io/io.stub lib/chibi/io/io.c)
chibi_ffi(lib/chibi/emscripten.stub lib/chibi/emscripten.c)
chibi_ffi(lib/chibi/net.stub lib/chibi/net.c)
chibi_ffi(lib/chibi/filesystem.stub lib/chibi/filesystem.c)
chibi_ffi(lib/chibi/stty.stub lib/chibi/stty.c)

chibi_module(crypto lib/chibi/crypto/crypto.c lib/chibi/crypto/ crypto)
chibi_module(process lib/chibi/process.c lib/chibi/ process)
chibi_module(system lib/chibi/system.c lib/chibi/ system)
chibi_module(chibi-time lib/chibi/time.c lib/chibi/ time)
chibi_module(io lib/chibi/io/io.c lib/chibi/io/ io)
chibi_module(emscripten lib/chibi/emscripten.c lib/chibi/ emscripten)
chibi_module(net lib/chibi/net.c lib/chibi/ net)
chibi_module(filesystem lib/chibi/filesystem.c lib/chibi/ filesystem)
chibi_module(stty lib/chibi/stty.c lib/chibi/ stty)
    
chibi_module(weak lib/chibi/weak.c lib/chibi/ weak)
chibi_module(heap-stats lib/chibi/heap-stats.c lib/chibi/ heap-stats)
chibi_module(disasm lib/chibi/disasm.c lib/chibi/ disasm)
chibi_module(ast lib/chibi/ast.c lib/chibi/ ast)

chibi_module(rest lib/chibi/optimize/rest.c lib/chibi/optimize/ rest)

chibi_module(threads lib/srfi/18/threads.c lib/srfi/18/ threads)
chibi_module(rand lib/srfi/27/rand.c lib/srfi/27/ rand)
chibi_module(bit lib/srfi/33/bit.c lib/srfi/33/ bit)
chibi_module(param lib/srfi/39/param.c lib/srfi/39/ param)
chibi_module(hash lib/srfi/69/hash.c lib/srfi/69/ hash)
chibi_module(qsort lib/srfi/95/qsort.c lib/srfi/95/ qsort)
chibi_module(env lib/srfi/98/env.c lib/srfi/98/ env)

chibi_module(scheme-time lib/scheme/time.c lib/scheme/ time)

add_custom_target(clibs.c
  COMMAND ${CHIBI_EXECUTABLE} -q ${PROJECT_SOURCE_DIR}/tools/chibi-genstatic < genstatic-input > include/clibs.c
  DEPENDS ${GENERATED_SOURCES})

if(${SEXP_USE_STATIC_LIBS})
  add_dependencies(chibi-scheme-lib clibs.c)
endif()

### Testing
enable_testing()
macro(chibi_test_simple name scm)
  add_test(NAME ${name}
    COMMAND $<TARGET_FILE:chibi-scheme> ${PROJECT_SOURCE_DIR}/${scm}
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR})

  set_tests_properties(${name} PROPERTIES ENVIRONMENT "CHIBI_MODULE_PATH=${PROJECT_BINARY_DIR}/$<CONFIG>/lib")
endmacro(chibi_test_simple)

chibi_test_simple(test-r7rs tests/r7rs-tests.scm)
chibi_test_simple(test-libs tests/lib-tests.scm)
