cmake_minimum_required(VERSION 2.8.11)
if("${CMAKE_CXX_COMPILER}" STREQUAL "")
       set(CMAKE_C_COMPILER "clang")
       set(CMAKE_CXX_COMPILER "dpcpp")
endif()
project (oneDNN_CNN)
if("$ENV{EXAMPLE_ROOT}" STREQUAL "")
	message(" - use default examples")
	add_subdirectory ($ENV{DNNLROOT}/examples out)
else()
	add_subdirectory ($ENV{EXAMPLE_ROOT} out)
endif()
