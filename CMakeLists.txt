cmake_minimum_required(VERSION 3.10)
project(fake_meminfo C)

# Always build as a shared library
add_library(fake_meminfo SHARED fake-meminfo.c)

# Set the output name to "fake_meminfo.so"
set_target_properties(fake_meminfo PROPERTIES PREFIX "" OUTPUT_NAME "fake_meminfo")

# Default build type to Release if none specified
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

# Link with dl (needed for dlsym/dlopen if used)
target_link_libraries(fake_meminfo dl)
