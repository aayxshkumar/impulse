cmake_minimum_required(VERSION 3.24)
project(impulse LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

option(IMPULSE_BUILD_TESTS "Build unit tests" ON)
# e.g. -DIMPULSE_SANITIZERS=address,undefined
set(IMPULSE_SANITIZERS "" CACHE STRING "Comma-separated -fsanitize= list")

if(IMPULSE_SANITIZERS)
  add_compile_options(-fsanitize=${IMPULSE_SANITIZERS} -fno-omit-frame-pointer -fno-sanitize-recover=all)
  add_link_options(-fsanitize=${IMPULSE_SANITIZERS})
endif()

# No fused multiply-add contraction: results then depend only on the source
# and libm, not on whether the compiler chose to fuse a*b+c.
add_compile_options(-ffp-contract=off)
set(IMPULSE_WARNINGS -Wall -Wextra -Wpedantic -Werror)

add_library(impulse_core src/shape.cpp src/collision.cpp src/world.cpp src/scenes.cpp)
target_include_directories(impulse_core PUBLIC include)
target_compile_options(impulse_core PRIVATE ${IMPULSE_WARNINGS})

add_executable(impulse_sim tools/impulse_sim.cpp)
target_link_libraries(impulse_sim PRIVATE impulse_core)
target_compile_options(impulse_sim PRIVATE ${IMPULSE_WARNINGS})

if(IMPULSE_BUILD_TESTS)
  include(FetchContent)
  # Vendored tarball when present (offline machines), else GitHub; same hash.
  set(_gtest_tarball ${CMAKE_CURRENT_SOURCE_DIR}/third_party/googletest-1.15.2.tar.gz)
  if(EXISTS ${_gtest_tarball})
    set(_gtest_url ${_gtest_tarball})
  else()
    set(_gtest_url https://github.com/google/googletest/archive/refs/tags/v1.15.2.tar.gz)
  endif()
  FetchContent_Declare(googletest
    URL ${_gtest_url}
    URL_HASH SHA256=7b42b4d6ed48810c5362c265a17faebe90dc2373c885e5216439d37927f02926
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE)
  set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
  set(BUILD_GMOCK OFF CACHE BOOL "" FORCE)
  FetchContent_MakeAvailable(googletest)

  enable_testing()
  add_executable(impulse_tests tests/shape_test.cpp tests/collision_test.cpp tests/world_test.cpp)
  target_link_libraries(impulse_tests PRIVATE impulse_core GTest::gtest_main)
  target_compile_options(impulse_tests PRIVATE ${IMPULSE_WARNINGS})
  include(GoogleTest)
  gtest_discover_tests(impulse_tests DISCOVERY_MODE PRE_TEST DISCOVERY_TIMEOUT 60)
  add_test(NAME physical_checks COMMAND impulse_sim --check)
endif()
