# -------------------- dependency target selection --------------------

# console_bridge target name
if(TARGET console_bridge::console_bridge)
  set(URDFDOM_CONSOLE_BRIDGE_TARGET console_bridge::console_bridge)
elseif(TARGET console_bridge)
  set(URDFDOM_CONSOLE_BRIDGE_TARGET console_bridge)
else()
  message(FATAL_ERROR "console_bridge target not found. Did you add_subdirectory(vendor/console_bridge) at top-level?")
endif()

# urdfdom_headers target name
if(TARGET urdfdom_headers::urdfdom_headers)
  set(URDFDOM_HEADERS_TARGET urdfdom_headers::urdfdom_headers)
elseif(TARGET urdfdom_headers)
  set(URDFDOM_HEADERS_TARGET urdfdom_headers)
else()
  message(FATAL_ERROR "urdfdom_headers target not found. Did you add_subdirectory(vendor/urdfdom_headers) at top-level?")
endif()

# tinyxml2 target name
if(TARGET tinyxml2::tinyxml2)
  set(URDFDOM_TINYXML2_TARGET tinyxml2::tinyxml2)
elseif(TARGET tinyxml2)
  set(URDFDOM_TINYXML2_TARGET tinyxml2)
else()
  message(FATAL_ERROR "tinyxml2 target not found. Did you add_subdirectory(vendor/tinyxml2) at top-level?")
endif()

# -------------------- helper macro --------------------

function(add_urdfdom_library)
  set(options)
  set(oneValueArgs LIBNAME)
  set(multiValueArgs SOURCES LINK)

  cmake_parse_arguments(ARG
    "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

  if(NOT ARG_LIBNAME)
    message(FATAL_ERROR "add_urdfdom_library: LIBNAME is required")
  endif()

  add_library(${ARG_LIBNAME} SHARED ${ARG_SOURCES})
  add_library(urdfdom::${ARG_LIBNAME} ALIAS ${ARG_LIBNAME})

  target_include_directories(${ARG_LIBNAME} PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  )

  # Base deps that every urdfdom shared lib needs.
  target_link_libraries(${ARG_LIBNAME} PUBLIC
    ${URDFDOM_HEADERS_TARGET}
    ${URDFDOM_CONSOLE_BRIDGE_TARGET}
    ${ARG_LINK}
  )

  # Recommended for this layout on Linux
  if(UNIX AND NOT APPLE)
    target_link_options(${ARG_LIBNAME} PRIVATE "LINKER:--no-undefined")
  endif()

  if(NOT CMAKE_CXX_STANDARD)
    target_compile_features(${ARG_LIBNAME} PUBLIC cxx_std_14)
  endif()

  set_target_properties(${ARG_LIBNAME} PROPERTIES
    DEFINE_SYMBOL URDFDOM_EXPORTS
    SOVERSION ${URDF_MAJOR_MINOR_VERSION}
  )
endfunction()

# -------------------- libraries --------------------

# Core model lib: this is where tinyxml2 is actually used.
add_urdfdom_library(
  LIBNAME urdfdom_model
  SOURCES
    src/pose.cpp
    src/model.cpp
    src/link.cpp
    src/joint.cpp
  LINK
    ${URDFDOM_TINYXML2_TARGET}
)

# World builds only world.cpp and reuses urdfdom_model
add_urdfdom_library(
  LIBNAME urdfdom_world
  SOURCES
    src/world.cpp
  LINK
    urdfdom::urdfdom_model
)

# Sensor builds only sensor code and reuses urdfdom_model
add_urdfdom_library(
  LIBNAME urdfdom_sensor
  SOURCES
    src/urdf_sensor.cpp
  LINK
    urdfdom::urdfdom_model
)

add_library(urdf_parser INTERFACE)
target_include_directories(urdf_parser INTERFACE
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
)
target_link_libraries(urdf_parser INTERFACE
  urdfdom::urdfdom_model
  urdfdom::urdfdom_sensor
  urdfdom::urdfdom_world
)

# -------------------- executables --------------------

add_executable(check_urdf src/check_urdf.cpp)
target_link_libraries(check_urdf PRIVATE urdfdom::urdfdom_model urdfdom::urdfdom_world)

add_executable(urdf_to_graphviz src/urdf_to_graphviz.cpp)
target_link_libraries(urdf_to_graphviz PRIVATE urdfdom::urdfdom_model)

add_executable(urdf_mem_test test/memtest.cpp)
target_link_libraries(urdf_mem_test PRIVATE urdfdom::urdfdom_model)

include(CTest)
if(BUILD_TESTING)
  add_subdirectory(test)
endif()




# macro(add_urdfdom_library)
#   set(oneValueArgs LIBNAME)
#   set(multiValueArgs SOURCES LINK)
#   cmake_parse_arguments(add_urdfdom_library "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

#   add_library(${add_urdfdom_library_LIBNAME} SHARED
#     ${add_urdfdom_library_SOURCES})
#   add_library(urdfdom::${add_urdfdom_library_LIBNAME} ALIAS ${add_urdfdom_library_LIBNAME})
#   target_include_directories(${add_urdfdom_library_LIBNAME} PUBLIC
#     "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>")
#  #   "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
#   target_link_libraries(${add_urdfdom_library_LIBNAME} PUBLIC
#     ${add_urdfdom_library_LINK}
#     ${urdfdom_headers_link_libs}
#   )
#   target_link_libraries(${add_urdfdom_library_LIBNAME} PRIVATE
#     ${console_bridge_link_libs}
#     tinyxml2::tinyxml2
#     urdfdom_headers
#     console_bridge
#   )

#     # Optionally force resolution now (helps avoid undefined symbols leaking)
#   if(UNIX AND NOT APPLE)
#     target_link_options(${name} PRIVATE "LINKER:--no-undefined")
#   endif()

#   ########################

#   # Force shared libs to resolve all symbols now (avoid undefined refs leaking to executables)
# foreach(t urdfdom_model urdfdom_world)
#   if(TARGET ${t})
#     target_link_options(${t} PRIVATE "LINKER:--no-undefined")
#   endif()
# endforeach()


# # Make urdf_model/*.h visible to the parser libs
# foreach(t urdfdom_model urdfdom_world)
#   if(TARGET ${t})
#     target_link_libraries(${t} PUBLIC urdfdom_headers)
#   endif()
# endforeach()

# # Make sure the shared libs are fully linked (no undefined symbols)
# foreach(t urdfdom_model urdfdom_world urdfdom_sensor)
#   if(TARGET ${t})
#     target_link_libraries(${t} PUBLIC
#       tinyxml2::tinyxml2           # or tinyxml2::tinyxml2 if you created the alias
#       console_bridge     # target from your submodule
#       urdfdom_headers    # interface headers target
#     )
#   endif()
# endforeach()

# target_link_libraries(urdfdom_model PUBLIC tinyxml2 console_bridge urdfdom_headers)
# target_link_libraries(urdfdom_world PUBLIC tinyxml2 console_bridge urdfdom_headers)

# ########################


#   if(NOT CMAKE_CXX_STANDARD)
#     target_compile_features(${add_urdfdom_library_LIBNAME} PUBLIC cxx_std_14)
#   endif()
#   set_target_properties(${add_urdfdom_library_LIBNAME} PROPERTIES
#     DEFINE_SYMBOL URDFDOM_EXPORTS
#     SOVERSION ${URDF_MAJOR_MINOR_VERSION})
# endmacro()

# if(TARGET console_bridge::console_bridge)
#   set(console_bridge_link_libs "console_bridge::console_bridge")
# else()
#   set(console_bridge_link_libs "console_bridge")
# endif()

# if(TARGET urdfdom_headers::urdfdom_headers)
#   set(urdfdom_headers_link_libs "urdfdom_headers::urdfdom_headers")
# else()
#   set(urdfdom_headers_link_libs "")
# endif()

# add_urdfdom_library(
#   LIBNAME
#     urdfdom_world
#   SOURCES
#     src/pose.cpp
#     src/model.cpp
#     src/link.cpp
#     src/joint.cpp
#     src/world.cpp)

# add_urdfdom_library(
#   LIBNAME
#     urdfdom_model
#   SOURCES
#     src/pose.cpp
#     src/model.cpp
#     src/link.cpp
#     src/joint.cpp)

# add_urdfdom_library(
#   LIBNAME
#     urdfdom_sensor
#   SOURCES
#     src/urdf_sensor.cpp
#   LINK
#     urdfdom_model)

# add_library(urdf_parser INTERFACE)
# target_include_directories(urdf_parser INTERFACE
#   "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>")
# #  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
# target_link_libraries(urdf_parser INTERFACE
#   urdfdom::urdfdom_model
#   urdfdom::urdfdom_sensor
#   urdfdom::urdfdom_world)

# # --------------------------------

# add_executable(check_urdf src/check_urdf.cpp)
# target_include_directories(check_urdf PUBLIC include)
# target_link_libraries(check_urdf urdfdom_model urdfdom_world)

# add_executable(urdf_to_graphviz src/urdf_to_graphviz.cpp)
# target_include_directories(urdf_to_graphviz PUBLIC include)
# target_link_libraries(urdf_to_graphviz urdfdom_model)

# # urdf_mem_test is a binary for testing, not a unit test
# add_executable(urdf_mem_test test/memtest.cpp)
# target_include_directories(urdf_mem_test PUBLIC include)
# target_link_libraries(urdf_mem_test urdfdom_model)

# include(CTest)
# if(BUILD_TESTING)
#   # TODO: check Shane's comment https://github.com/ros/urdfdom/pull/157/files#r664960227
#   add_subdirectory(test)
# endif()

# #INSTALL(
# #  TARGETS
# #  check_urdf
# #  urdf_to_graphviz
# #  urdf_mem_test
# #  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
# #  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
# #  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
# #)
# #INSTALL(
# #  TARGETS
# #  urdfdom_model
# #  urdfdom_world
# #  urdfdom_sensor
# #  urdf_parser
# #  EXPORT
# #  urdfdom
# #  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
# #  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
# #  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
# #)
# #install(
# #  EXPORT urdfdom
# #  DESTINATION ${CMAKE_CONFIG_INSTALL_DIR}
# #  NAMESPACE "${PROJECT_NAME}::"
# #  FILE "urdfdomExport.cmake"
# #)

# #INSTALL(DIRECTORY include/urdf_parser DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
