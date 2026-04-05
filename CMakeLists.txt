cmake_minimum_required(VERSION 3.16)
project(threaded_image_ops LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

include(FetchContent)

FetchContent_Declare(
    pybind11
    GIT_REPOSITORY https://github.com/pybind/pybind11.git
    GIT_TAG v3.0.2
)
FetchContent_MakeAvailable(pybind11)

FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.17.0
)
FetchContent_MakeAvailable(googletest)

pybind11_add_module(threaded_image_ops src/module.cpp src/transforms.cpp)
target_include_directories(threaded_image_ops PRIVATE ${CMAKE_SOURCE_DIR}/include)

enable_testing()
include(GoogleTest)

add_executable(test_transforms tests/test_transform_quadrant.cpp)
target_include_directories(test_transforms PRIVATE ${CMAKE_SOURCE_DIR}/include)
target_link_libraries(test_transforms GTest::gtest_main)
gtest_discover_tests(test_transforms)

add_executable(test_partitioning tests/test_quadrant_partitioning.cpp)
target_include_directories(test_partitioning PRIVATE ${CMAKE_SOURCE_DIR}/include)
target_link_libraries(test_partitioning GTest::gtest_main)
gtest_discover_tests(test_partitioning)

add_executable(test_thread_safety tests/test_thread_safety.cpp)
target_include_directories(test_thread_safety PRIVATE ${CMAKE_SOURCE_DIR}/include)
find_package(Threads REQUIRED)
target_link_libraries(test_thread_safety GTest::gtest_main Threads::Threads)
gtest_discover_tests(test_thread_safety)
