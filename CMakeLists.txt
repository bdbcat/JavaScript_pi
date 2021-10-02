##---------------------------------------------------------------------------
## Author:      Tony Voss
## Copyright:
## License:     wxWidgets License
##---------------------------------------------------------------------------

# -------- Cmake setup ---------
#
cmake_minimum_required(VERSION 3.12.0)

cmake_policy(SET CMP0042 NEW)

if (POLICY CMP0072)
  cmake_policy(SET CMP0072 NEW)
endif ()

if (POLICY CMP0077)
  cmake_policy(SET CMP0077 NEW)
endif ()

# Locations where cmake looks for cmake modules.
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/build ${CMAKE_SOURCE_DIR}/cmake)

if (WIN32)
  list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/buildwin)
endif ()
message(STATUS "Cmake version: ${CMAKE_VERSION}.")


# -------- Build setup (mostly in PluginCompiler) --------
#
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
message(STATUS "Building: ${BUILD_TYPE}")

# -------- Options ----------
set(OCPN_TEST_REPO
    "opencpn/javascript-alpha"
    CACHE STRING "Default repository for untagged builds"
)
set(OCPN_BETA_REPO
    "opencpn/javascript-beta"
    CACHE STRING 
    "Default repository for tagged builds matching 'beta'"
)
set(OCPN_RELEASE_REPO
    "opencpn/javascript-prod"
    CACHE STRING 
    "Default repository for tagged builds not matching 'beta'"
)
option(JAVASCRIPT_USE_SVG "Use SVG graphics" ON)

set(OCPN_TARGET_TUPLE "" CACHE STRING
  "Target spec: \"platform;version;arch\""
)

string(TOLOWER "${OCPN_TARGET_TUPLE}" _lc_target)
if ("${_lc_target}" MATCHES "android*")
  set(QT_ANDROID ON)
  # Until we have a proper toolchain file:
  set(CMAKE_CROSSCOMPILING ON)
else ()
  set(QT_ANDROID OFF)
  add_definitions(-D__OCPN_USE_CURL__)
endif ()
#
#
# -------  Plugin setup --------
#
project(javascript_pi VERSION 0.4.0.11)
set(PKG_PRERELEASE "")  # Empty, or a tag like 'beta'

set(DISPLAY_NAME JavaScript)    # Dialogs, installer artifacts, ...
set(PLUGIN_API_NAME JavaScript) # As of GetCommonName() in plugin API
set(CPACK_PACKAGE_CONTACT "Mike Rossiter")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Simulate ship movements")
set(CPACK_PACKAGE_DESCRIPTION [=[
This plugin for OpenCPN allows you to run JavaScript and to interact with OpenCPN.
You can use it to script your own enhancements to standard OpenCPN functionality.
]=])

set(PKG_AUTHOR "Tony Voss")
set(PKG_IS_OPEN_SOURCE "yes")
set(CPACK_PACKAGE_HOMEPAGE_URL https://github.com/antipole2/javascript_pi)
set(PKG_INFO_URL https://opencpn.org/OpenCPN/plugins/javascript.html)


SET(SRC
    src/version.h
    src/trace.h
    src/JavaScript_pi.h
    src/JavaScript_pi.cpp
    src/icons.h
    src/icons.cpp
    src/JavaScriptgui.h
    src/JavaScriptgui.cpp
    src/JavaScriptgui_impl.cpp
    src/JavaScriptgui_impl.h
    src/JSExtensions.cpp
    src/ocpn_duk.h
    src/OPCNapis.cpp
    src/duk_config.h
    src/duktape.h
    src/duktape.cpp
    src/JSlexer.cpp
    src/JSdialog.cpp
    src/functions.cpp
    src/optional.cpp
    src/toolsDialogGui.h
    src/toolsDialogGui.cpp
    src/toolsDialogImp.h
    src/toolsDialogImp.cpp
    )
    
add_library(${CMAKE_PROJECT_NAME} SHARED ${SRC})

ADD_DEFINITIONS(-DSCI_LEXER)

# -------- Setup completed, build the plugin --------
#

include_directories(BEFORE ${CMAKE_BINARY_DIR}/include)

add_subdirectory("libs/api-16")
target_link_libraries(${CMAKE_PROJECT_NAME} ocpn::api)

set(VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(VERSION_PATCH ${PROJECT_VERSION_PATCH})

set(PACKAGE_NAME ${CMAKE_PROJECT_NAME})

# Create the minimim ocpn version id string like ov50
string(REGEX REPLACE "\([0-9]\)\.\([0-9]\).*" "ov\\1\\2"
       OCPN_MIN_VERSION ${API_OCPN_MIN_VERSION})

if (JAVASCRIPT_USE_SVG)
  add_definitions(-DJAVASCRIPT_USE_SVG)
endif ()

# Set up targets. Targets sets up a recursive call with BUILD_TYPE set to
# 'flatpak', 'pkg' or 'tarball'. The initial call without BUILD_TYPE ends
# here.
#
include(Targets)
create_targets(
  ${PROJECT_SOURCE_DIR}/flatpak/org.opencpn.OpenCPN.Plugin.javascript.yaml
)
if ("${BUILD_TYPE}" STREQUAL "")
  return ()
endif ()

if (NOT ${BUILD_TYPE} STREQUAL "flatpak")
  # Build package as required (flatpak already dealt with).
  #
  include(PluginCompiler)
  include(PluginInstall)
  include(PluginPackage)
  if (QT_ANDROID)
    include(AndroidLibs)
  else ()
    include(PluginLibs)
  endif ()
  include(PluginLocalization)

  include_directories(BEFORE ${CMAKE_BINARY_DIR}/include)

  add_subdirectory("libs/jsoncpp")
  target_link_libraries(${PACKAGE_NAME} ocpn::jsoncpp)
  
  add_subdirectory("libs/tinyxml")
  target_link_libraries(${PACKAGE_NAME} ocpn::tinyxml)
  
  add_subdirectory("libs/wxJSON")
  target_link_libraries(${PACKAGE_NAME} ocpn::wxjson)

endif ()

configure_file(
  ${CMAKE_SOURCE_DIR}/config.h.in ${CMAKE_BINARY_DIR}/include//config.h
  @ONLY
)

configure_file(
  # The cloudsmith upload script
  ${CMAKE_SOURCE_DIR}/ci/upload.sh.in ${CMAKE_BINARY_DIR}/upload.sh
  @ONLY
)
configure_file(
  # The cloudsmith upload script, windows bat file.
  ${CMAKE_SOURCE_DIR}/ci/upload.bat.in ${CMAKE_BINARY_DIR}/upload.bat
  @ONLY
)


set(checksum "@checksum@")
configure_file(
  # The XML metadata file
  ${CMAKE_SOURCE_DIR}/plugin.xml.in
  ${CMAKE_BINARY_DIR}/${pkg_displayname}.xml.in
  @ONLY
)

