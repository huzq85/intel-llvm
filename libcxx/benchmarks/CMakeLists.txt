if (CMAKE_VERSION VERSION_LESS 3.17)
  message(WARNING "The libc++ benchmarks won't be available because the version of CMake is too old to support them.")
  return()
endif()

include(ExternalProject)
include(CheckCXXCompilerFlag)

#==============================================================================
# Build Google Benchmark for libc++
#==============================================================================

set(CMAKE_FOLDER "${CMAKE_FOLDER}/Benchmarks")

set(BENCHMARK_LIBCXX_COMPILE_FLAGS
    -Wno-unused-command-line-argument
    -nostdinc++
    -isystem "${LIBCXX_GENERATED_INCLUDE_DIR}"
    -L${LIBCXX_LIBRARY_DIR}
    -Wl,-rpath,${LIBCXX_LIBRARY_DIR}
    ${SANITIZER_FLAGS}
    )
if(LLVM_ENABLE_PER_TARGET_RUNTIME_DIR AND NOT APPLE)
  list(APPEND BENCHMARK_LIBCXX_COMPILE_FLAGS
    -isystem "${LIBCXX_GENERATED_INCLUDE_TARGET_DIR}")
endif()
if (DEFINED LIBCXX_CXX_ABI_LIBRARY_PATH)
  list(APPEND BENCHMARK_LIBCXX_COMPILE_FLAGS
          -L${LIBCXX_CXX_ABI_LIBRARY_PATH}
          -Wl,-rpath,${LIBCXX_CXX_ABI_LIBRARY_PATH})
endif()
split_list(BENCHMARK_LIBCXX_COMPILE_FLAGS)

ExternalProject_Add(google-benchmark-libcxx
        EXCLUDE_FROM_ALL ON
        DEPENDS cxx cxx-headers
        PREFIX benchmark-libcxx
        SOURCE_DIR ${LLVM_THIRD_PARTY_DIR}/benchmark
        INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/benchmark-libcxx
        CMAKE_CACHE_ARGS
          -DCMAKE_C_COMPILER:STRING=${CMAKE_C_COMPILER}
          -DCMAKE_CXX_COMPILER:STRING=${CMAKE_CXX_COMPILER}
          -DCMAKE_BUILD_TYPE:STRING=RELEASE
          -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
          -DCMAKE_CXX_FLAGS:STRING=${BENCHMARK_LIBCXX_COMPILE_FLAGS}
          -DBENCHMARK_USE_LIBCXX:BOOL=ON
          -DBENCHMARK_ENABLE_TESTING:BOOL=OFF)

#==============================================================================
# Build Google Benchmark for the native stdlib
#==============================================================================
set(BENCHMARK_NATIVE_TARGET_FLAGS)
if (LIBCXX_BENCHMARK_NATIVE_GCC_TOOLCHAIN)
  set(BENCHMARK_NATIVE_TARGET_FLAGS
      --gcc-toolchain=${LIBCXX_BENCHMARK_NATIVE_GCC_TOOLCHAIN})
endif()
split_list(BENCHMARK_NATIVE_TARGET_FLAGS)

if (LIBCXX_BENCHMARK_NATIVE_STDLIB)
  ExternalProject_Add(google-benchmark-native
        EXCLUDE_FROM_ALL ON
        PREFIX benchmark-native
        SOURCE_DIR ${LLVM_THIRD_PARTY_DIR}/benchmark
        INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/benchmark-native
        CMAKE_CACHE_ARGS
          -DCMAKE_C_COMPILER:STRING=${CMAKE_C_COMPILER}
          -DCMAKE_CXX_COMPILER:STRING=${CMAKE_CXX_COMPILER}
          -DCMAKE_CXX_FLAGS:STRING=${BENCHMARK_NATIVE_TARGET_FLAGS}
          -DCMAKE_BUILD_TYPE:STRING=RELEASE
          -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
          -DBENCHMARK_ENABLE_TESTING:BOOL=OFF)
endif()


#==============================================================================
# Benchmark tests configuration
#==============================================================================
add_custom_target(cxx-benchmarks)
set(BENCHMARK_OUTPUT_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(BENCHMARK_LIBCXX_INSTALL ${CMAKE_CURRENT_BINARY_DIR}/benchmark-libcxx)
set(BENCHMARK_NATIVE_INSTALL ${CMAKE_CURRENT_BINARY_DIR}/benchmark-native)

add_library(               cxx-benchmarks-flags INTERFACE)
target_compile_features(   cxx-benchmarks-flags INTERFACE cxx_std_20)
target_compile_options(    cxx-benchmarks-flags INTERFACE -fsized-deallocation -nostdinc++)
target_include_directories(cxx-benchmarks-flags INTERFACE "${LIBCXX_GENERATED_INCLUDE_DIR}"
                                                INTERFACE "${BENCHMARK_LIBCXX_INSTALL}/include"
                                                INTERFACE "${LIBCXX_SOURCE_DIR}/test/support")

add_library(           cxx-benchmarks-flags-native INTERFACE)
target_link_libraries( cxx-benchmarks-flags-native INTERFACE cxx-benchmarks-flags)
target_compile_options(cxx-benchmarks-flags-native INTERFACE ${BENCHMARK_NATIVE_TARGET_FLAGS})
target_link_options(   cxx-benchmarks-flags-native INTERFACE ${BENCHMARK_NATIVE_TARGET_FLAGS} "-L${BENCHMARK_NATIVE_INSTALL}/lib")
if (LIBCXX_BENCHMARK_NATIVE_STDLIB STREQUAL "libstdc++")
  find_library(LIBSTDCXX_FILESYSTEM_TEST stdc++fs
        PATHS ${LIBCXX_BENCHMARK_NATIVE_GCC_TOOLCHAIN}
        PATH_SUFFIXES lib lib64
        DOC "The libstdc++ filesystem library used by the benchmarks"
    )
  if (LIBSTDCXX_FILESYSTEM_TEST)
    target_link_libraries(cxx-benchmarks-flags-native INTERFACE -lstdc++fs)
  endif()
else()
  target_link_libraries(cxx-benchmarks-flags-native INTERFACE -lc++fs -lc++experimental)
endif()

add_library(           cxx-benchmarks-flags-libcxx INTERFACE)
target_link_libraries( cxx-benchmarks-flags-libcxx INTERFACE cxx-benchmarks-flags)
target_compile_options(cxx-benchmarks-flags-libcxx INTERFACE ${SANITIZER_FLAGS} -Wno-user-defined-literals -Wno-suggest-override)
target_link_options(   cxx-benchmarks-flags-libcxx INTERFACE -nodefaultlibs "-L${BENCHMARK_LIBCXX_INSTALL}/lib" "-L${BENCHMARK_LIBCXX_INSTALL}/lib64" ${SANITIZER_FLAGS})

set(libcxx_benchmark_targets)

function(add_benchmark_test name source_file)
  set(libcxx_target ${name}_libcxx)
  list(APPEND libcxx_benchmark_targets ${libcxx_target})
  add_executable(${libcxx_target} EXCLUDE_FROM_ALL ${source_file})
  target_link_libraries(${libcxx_target} PRIVATE cxx-benchmarks-flags-libcxx)
  add_dependencies(${libcxx_target} cxx google-benchmark-libcxx)
  add_dependencies(cxx-benchmarks ${libcxx_target})
  if (LIBCXX_ENABLE_SHARED)
    target_link_libraries(${libcxx_target} PRIVATE cxx_shared)
  else()
    target_link_libraries(${libcxx_target} PRIVATE cxx_static)
  endif()
  target_link_libraries(${libcxx_target} PRIVATE cxx_experimental benchmark)
  if (LLVM_USE_SANITIZER)
    target_link_libraries(${libcxx_target} PRIVATE -ldl)
  endif()
  set_target_properties(${libcxx_target}
    PROPERTIES
          OUTPUT_NAME "${name}.libcxx.out"
          RUNTIME_OUTPUT_DIRECTORY "${BENCHMARK_OUTPUT_DIR}"
          CXX_EXTENSIONS NO)
  cxx_link_system_libraries(${libcxx_target})
  if (LIBCXX_BENCHMARK_NATIVE_STDLIB)
    set(native_target ${name}_native)
    add_executable(${native_target} EXCLUDE_FROM_ALL ${source_file})
    target_link_libraries(${native_target} PRIVATE cxx-benchmarks-flags-native)
    add_dependencies(${native_target} google-benchmark-native
                                      google-benchmark-libcxx)
    target_link_libraries(${native_target} PRIVATE -lbenchmark)
    if (LIBCXX_HAS_PTHREAD_LIB)
      target_link_libraries(${native_target} PRIVATE -pthread)
    endif()
    add_dependencies(cxx-benchmarks ${native_target})
    set_target_properties(${native_target}
      PROPERTIES
          OUTPUT_NAME "${name}.native.out"
          RUNTIME_OUTPUT_DIRECTORY "${BENCHMARK_OUTPUT_DIR}"
          CXX_EXTENSIONS NO)
  endif()
endfunction()


#==============================================================================
# Register Benchmark tests
#==============================================================================
set(BENCHMARK_TESTS
    algorithms.partition_point.bench.cpp
    algorithms/equal.bench.cpp
    algorithms/find.bench.cpp
    algorithms/lower_bound.bench.cpp
    algorithms/make_heap.bench.cpp
    algorithms/make_heap_then_sort_heap.bench.cpp
    algorithms/min.bench.cpp
    algorithms/min_max_element.bench.cpp
    algorithms/pop_heap.bench.cpp
    algorithms/push_heap.bench.cpp
    algorithms/ranges_make_heap.bench.cpp
    algorithms/ranges_make_heap_then_sort_heap.bench.cpp
    algorithms/ranges_pop_heap.bench.cpp
    algorithms/ranges_push_heap.bench.cpp
    algorithms/ranges_sort.bench.cpp
    algorithms/ranges_sort_heap.bench.cpp
    algorithms/ranges_stable_sort.bench.cpp
    algorithms/sort.bench.cpp
    algorithms/sort_heap.bench.cpp
    algorithms/stable_sort.bench.cpp
    libcxxabi/dynamic_cast.bench.cpp
    allocation.bench.cpp
    deque.bench.cpp
    deque_iterator.bench.cpp
    filesystem.bench.cpp
    format_to_n.bench.cpp
    format_to.bench.cpp
    format.bench.cpp
    formatted_size.bench.cpp
    formatter_float.bench.cpp
    formatter_int.bench.cpp
    function.bench.cpp
    join_view.bench.cpp
    lexicographical_compare_three_way.bench.cpp
    map.bench.cpp
    monotonic_buffer.bench.cpp
    ordered_set.bench.cpp
    std_format_spec_string_unicode.bench.cpp
    string.bench.cpp
    stringstream.bench.cpp
    to_chars.bench.cpp
    unordered_set_operations.bench.cpp
    util_smartptr.bench.cpp
    variant_visit_1.bench.cpp
    variant_visit_2.bench.cpp
    variant_visit_3.bench.cpp
    vector_operations.bench.cpp
    )

foreach(test_path ${BENCHMARK_TESTS})
  get_filename_component(test_file "${test_path}" NAME)
  string(REPLACE ".bench.cpp" "" test_name "${test_file}")
  if (NOT DEFINED ${test_name}_REPORTED)
    message(STATUS "Adding Benchmark: ${test_file}")
    # Only report the adding of the benchmark once.
    set(${test_name}_REPORTED ON CACHE INTERNAL "")
  endif()
  add_benchmark_test(${test_name} ${test_path})
endforeach()

if (LIBCXX_INCLUDE_TESTS)
  include(AddLLVM)

  if (NOT DEFINED LIBCXX_TEST_DEPS)
    message(FATAL_ERROR "Expected LIBCXX_TEST_DEPS to be defined")
  endif()

  configure_lit_site_cfg(
          ${CMAKE_CURRENT_SOURCE_DIR}/lit.site.cfg.py.in
          ${CMAKE_CURRENT_BINARY_DIR}/lit.site.cfg.py)

  set(BENCHMARK_LIT_ARGS "--show-all --show-xfail --show-unsupported ${LIT_ARGS_DEFAULT}")

  add_lit_target(check-cxx-benchmarks
          "Running libcxx benchmarks tests"
          ${CMAKE_CURRENT_BINARY_DIR}
          DEPENDS cxx-benchmarks ${LIBCXX_TEST_DEPS}
          ARGS ${BENCHMARK_LIT_ARGS})
endif()
