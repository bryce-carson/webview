cmake_minimum_required(VERSION 3.12)
project(webview LANGUAGES C CXX)
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON) # enable position-independent code for shared libraries

R_INCLUDE_DIR)

# Find SWIG
find_package(SWIG 4.0 REQUIRED)
include(${SWIG_USE_FILE})

# Use webview headers from Git submodule
set(WEBVIEW_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/webview/core/include)
if(NOT EXISTS "${WEBVIEW_INCLUDE_DIR}/webview.h")
  message(FATAL_ERROR "webview.h not found in ${WEBVIEW_INCLUDE_DIR}. Did you run the initialize the Git submodule in the src/ subdirectory?")
endif()

# Set SWIG flags for R
set(CMAKE_SWIG_FLAGS -r -o "${CMAKE_CURRENT_BINARY_DIR}/R_wrap.c")

# Include directories for R and webview
include_directories(${R_INCLUDE_DIR} ${WEBVIEW_INCLUDE_DIR})

# Define the SWIG interface file
set(SWIG_INTERFACE webview/webview.i)

# SWIG source
set_source_files_properties(${SWIG_INTERFACE} PROPERTIES CPLUSPLUS ON)
swig_add_library(webview
  LANGUAGE r
  TYPE SHARED
  SOURCES
    ${SWIG_INTERFACE}
    ${CMAKE_CURRENT_SOURCE_DIR}/webview/core/src/webview.cc
)

# Link libraries (R libraries)
target_link_libraries(webview PRIVATE ${R_LIBRARY_FLAGS})

# Platform-specific linking for webview/webview
if(WIN32)
  target_link_libraries(webview PRIVATE advapi32 ole32 shell32 shlwapi user32 version)
elseif(UNIX AND NOT APPLE)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(GTK4 REQUIRED gtk4)
  pkg_check_modules(WEBKITGTK REQUIRED webkitgtk-6.0)
  target_include_directories(webview PRIVATE ${GTK4_INCLUDE_DIRS} ${WEBKITGTK_INCLUDE_DIRS})
  target_link_libraries(webview PRIVATE ${GTK4_LIBRARIES} ${WEBKITGTK_LIBRARIES} dl)
elseif(APPLE)
  target_link_libraries(webview PRIVATE "-framework WebKit")
endif()

# Set output name to match R package expectations
set_target_properties(webview PROPERTIES PREFIX "" OUTPUT_NAME "webview")

# Install the shared library to the R package's libs directory
install(TARGETS webview
  LIBRARY DESTINATION ${CMAKE_SOURCE_DIR}/../libs
  ARCHIVE DESTINATION ${CMAKE_SOURCE_DIR}/../libs
  RUNTIME DESTINATION ${CMAKE_SOURCE_DIR}/../libs  # For Windows DLLs
)

install(FILES "${CMAKE_SOURCE_DIR}/build/webview.R" DESTINATION ${CMAKE_SOURCE_DIR}/../R)
