cmake_minimum_required(VERSION 3.16...3.21)

project(obs-ndi VERSION 4.11.1)
add_library(${PROJECT_NAME} MODULE)

set(PLUGIN_AUTHOR "Stephane Lepin")
set(MACOS_BUNDLEID "fr.palakis.${PROJECT_NAME}")
set(LINUX_MAINTAINER_EMAIL "stephane.lepin@gmail.com")

if(UNIX AND NOT APPLE)
  set(OS_LINUX TRUE)
endif()

# Add your custom source files here - header files are optional and only required for visibility
# e.g. in Xcode or Visual Studio
target_sources(
  ${PROJECT_NAME}
  PRIVATE src/obs-ndi.cpp
          src/obs-ndi.cpp
          src/obs-ndi-source.cpp
          src/obs-ndi-output.cpp
          src/obs-ndi-filter.cpp
          src/premultiplied-alpha-filter.cpp
          src/main-output.cpp
          src/preview-output.cpp
          src/Config.cpp
          src/forms/output-settings.cpp
          src/obs-ndi.h
          src/main-output.h
          src/preview-output.h
          src/Config.h
          src/forms/output-settings.h)

# target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_SOURCE_DIR}/lib/ndi)
if(OS_LINUX AND NOT BUILD_NDI)
  target_include_directories(${PROJECT_NAME}
                            PRIVATE ${CMAKE_SOURCE_DIR}/lib/ndi)
else()
  target_include_directories(${PROJECT_NAME}
                            PRIVATE ${CMAKE_SOURCE_DIR}/plugins/obs-ndi/lib/ndi
                                    ${CMAKE_SOURCE_DIR}/UI/obs-frontend-api)
endif()

# Import libobs as main plugin dependency
if(OS_LINUX AND NOT BUILD_NDI)
  # Build obs-ndi plugin separately from OBS
  list(APPEND CMAKE_PREFIX_PATH "${CMAKE_SOURCE_DIR}/../../install/lib/cmake/libobs")
  find_package(libobs REQUIRED)
  include(cmake/ObsPluginHelpers.cmake)
endif()

if(OS_LINUX AND NOT BUILD_NDI)
  list(APPEND CMAKE_PREFIX_PATH "${CMAKE_SOURCE_DIR}/../../install/lib/cmake/obs-frontend-api")
else()
  list(APPEND CMAKE_PREFIX_PATH "${CMAKE_BINARY_DIR}/UI/obs-frontend-api")
endif()
find_package(obs-frontend-api REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE OBS::obs-frontend-api)

find_qt(COMPONENTS Widgets Core)
target_link_libraries(${PROJECT_NAME} PRIVATE Qt::Core Qt::Widgets)
set_target_properties(
  ${PROJECT_NAME}
  PROPERTIES AUTOMOC ON
             AUTOUIC ON
             AUTORCC ON)

if(OS_LINUX AND NOT BUILD_NDI)
  configure_file(src/plugin-macros.h.in ${CMAKE_SOURCE_DIR}/src/plugin-macros.generated.h)
else()
  configure_file(src/plugin-macros.h.in ${CMAKE_SOURCE_DIR}/plugins/obs-ndi/src/plugin-macros.generated.h)
endif()

target_sources(${PROJECT_NAME} PRIVATE src/plugin-macros.generated.h)

# /!\ TAKE NOTE: No need to edit things past this point /!\

# --- Platform-independent build settings ---

target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_SOURCE_DIR}/src)

target_link_libraries(${PROJECT_NAME} PRIVATE OBS::libobs)

# --- End of section ---

# --- Windows-specific build settings and tasks ---
if(OS_WINDOWS)
  configure_file(cmake/bundle/windows/installer-Windows.iss.in
                 ${CMAKE_BINARY_DIR}/installer-Windows.generated.iss)

  configure_file(cmake/bundle/windows/resource.rc.in ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.rc)
  target_sources(${PROJECT_NAME} PRIVATE ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.rc)

  if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /W4)
  endif()
  # --- End of section ---

  # -- macOS specific build settings and tasks --
elseif(OS_MACOS)
  configure_file(cmake/bundle/macos/installer-macos.pkgproj.in
                 ${CMAKE_BINARY_DIR}/installer-macos.generated.pkgproj)

  set(MACOSX_PLUGIN_GUI_IDENTIFIER "${MACOS_BUNDLEID}")
  set(MACOSX_PLUGIN_BUNDLE_VERSION "${CMAKE_PROJECT_VERSION}")
  set(MACOSX_PLUGIN_SHORT_VERSION_STRING "1")

  target_compile_options(${PROJECT_NAME} PRIVATE -Wall)
  # --- End of section ---

  # --- Linux-specific build settings and tasks ---
else()
  target_compile_options(${PROJECT_NAME} PRIVATE -Wall)
endif()
# --- End of section ---

if(NOT OS_LINUX)
  string(REPLACE "-" "_" project_name_without_dash "${PROJECT_NAME}")
  cpack_add_component_group(${project_name_without_dash}_group
                            DISPLAY_NAME "obs-ndi plugin"
                            DESCRIPTION "Network A/V in OBS Studio with NewTek's NDI technology")
endif()

setup_plugin_target(${PROJECT_NAME})
