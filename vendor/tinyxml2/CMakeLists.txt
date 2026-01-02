cmake_minimum_required(VERSION 3.15)
project(tinyxml2_vendor LANGUAGES CXX)

add_library(tinyxml2 STATIC
  tinyxml2.cpp
)

add_library(tinyxml2::tinyxml2 ALIAS tinyxml2)

target_include_directories(tinyxml2
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}>
)

target_compile_features(tinyxml2 PUBLIC cxx_std_11)

# Always PIC for static lib (so it can be linked into shared libs)
set_target_properties(tinyxml2 PROPERTIES
  POSITION_INDEPENDENT_CODE ON
  OUTPUT_NAME tinyxml2
)

# Extra safety on GNU/Clang: force -fPIC
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  target_compile_options(tinyxml2 PRIVATE -fPIC)
endif()

