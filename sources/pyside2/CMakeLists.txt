include(cmake/Macros/icecc.cmake) # this must be the first line!

cmake_minimum_required(VERSION 3.1)
cmake_policy(VERSION 3.1)

# Don't ignore targets that do not exist, inside add_dependencies calls.
cmake_policy(SET CMP0046 NEW)

project(pysidebindings)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../cmake_helpers/
                      ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Macros/
                      ${CMAKE_MODULE_PATH})
include(helpers)

# Don't display "up-to-date / install" messages when installing, to reduce visual clutter.
if (QUIET_BUILD)
    set(CMAKE_INSTALL_MESSAGE NEVER)
endif()

# Override message not to display info messages when doing a quiet build.
if (QUIET_BUILD AND is_pyside2_superproject_build)
    function(message)
        list(GET ARGV 0 MessageType)
        if (MessageType STREQUAL FATAL_ERROR OR
            MessageType STREQUAL SEND_ERROR OR
            MessageType STREQUAL WARNING OR
            MessageType STREQUAL AUTHOR_WARNING)
                list(REMOVE_AT ARGV 0)
                _message(${MessageType} "${ARGV}")
        endif()
    endfunction()
endif()

set(PYSIDE_VERSION_FILE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/pyside_version.py")
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS
  ${PYSIDE_VERSION_FILE_PATH}
)
execute_process(
  COMMAND ${PYTHON_EXECUTABLE} "${PYSIDE_VERSION_FILE_PATH}"
  OUTPUT_VARIABLE PYSIDE_VERSION_OUTPUT
  ERROR_VARIABLE PYSIDE_VERSION_OUTPUT_ERROR
  OUTPUT_STRIP_TRAILING_WHITESPACE)
if (NOT PYSIDE_VERSION_OUTPUT)
    message(FATAL_ERROR "Could not identify PySide2 version. Error: ${PYSIDE_VERSION_OUTPUT_ERROR}")
endif()

list(GET PYSIDE_VERSION_OUTPUT 0 BINDING_API_MAJOR_VERSION)
list(GET PYSIDE_VERSION_OUTPUT 1 BINDING_API_MINOR_VERSION)
list(GET PYSIDE_VERSION_OUTPUT 2 BINDING_API_MICRO_VERSION)
# a - alpha, b - beta, rc - rc
list(GET PYSIDE_VERSION_OUTPUT 3 BINDING_API_PRE_RELEASE_VERSION_TYPE)
# the number of the pre release (alpha1, beta3, rc7, etc.)
list(GET PYSIDE_VERSION_OUTPUT 4 BINDING_API_PRE_RELEASE_VERSION)

if (WIN32)
    set(PATH_SEP "\;")
else()
    set(PATH_SEP ":")
endif()

find_package(Shiboken2 2.0.0 REQUIRED)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "${SHIBOKEN_BUILD_TYPE}" CACHE STRING "Build Type")
endif()

if (CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions("-DNDEBUG")
endif()

if (SHIBOKEN_PYTHON_LIMITED_API)
    message(STATUS "******************************************************")
    message(STATUS "** PySide2 Limited API enabled.")
    message(STATUS "******************************************************")
endif()

find_package(Qt5 5.12 REQUIRED COMPONENTS Core)
add_definitions(${Qt5Core_DEFINITIONS})

find_file(GL_H "gl.h" PATH_SUFFIXES "GL")
message("result:" "${GL_H}")
include(FindQt5Extra)

set(XVFB_EXEC "")
option(USE_XVFB "Uses xvfb-run with the unit tests to avoid QtGui tests popping windows on the screen." FALSE)
if(USE_XVFB)
    find_program(XVFB_RUN NAMES xvfb-run)
    if (NOT ${XVFB_RUN} MATCHES "XVFB_RUN-NOTFOUND")
        set(XVFB_EXEC ${XVFB_RUN} -a)
        message(STATUS "Using xvfb-run to perform QtGui tests.")
    endif()
endif()

option(BUILD_TESTS "Build tests." TRUE)
option(ENABLE_VERSION_SUFFIX "Used to use current version in suffix to generated files. This is used to allow multiples versions installed simultaneous." FALSE)
set(LIB_SUFFIX "" CACHE STRING "Define suffix of directory name (32/64)" )
set(LIB_INSTALL_DIR "lib${LIB_SUFFIX}" CACHE PATH "The subdirectory relative to the install prefix where libraries will be installed (default is /lib${LIB_SUFFIX})" FORCE)
if(CMAKE_HOST_APPLE)
    set(ALTERNATIVE_QT_INCLUDE_DIR "" CACHE PATH "Deprecated. CMake now finds the proper include dir itself.")
    set(OSX_USE_LIBCPP "OFF" CACHE BOOL "Explicitly link the libc++ standard library (useful for osx deployment targets lower than 10.9.")
    if(OSX_USE_LIBCPP)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
    endif()
endif()

# Force usage of the C++11 standard, without a silent fallback
# to C++98 if the compiler does not support C++11.
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Qt5: QT_INCLUDE_DIR does no longer exist.
# On Windows, macOS, and Linux it can be computed from Qt5Core_INCLUDE_DIRS, which contains
# a list of include directories. We take the first one.
message(STATUS "*** Qt5Core_INCLUDE_DIRS = ${Qt5Core_INCLUDE_DIRS}")
list(GET Qt5Core_INCLUDE_DIRS 0 QT_INCLUDE_DIR)

# On macOS, check if Qt is a framework build. This affects how include paths should be handled.
get_target_property(QtCore_is_framework Qt5::Core FRAMEWORK)

if (QtCore_is_framework)
    # Get the path to the framework dir.
    get_filename_component(QT_FRAMEWORK_INCLUDE_DIR "${QT_INCLUDE_DIR}/../" ABSOLUTE)
    message(STATUS "*** QT_FRAMEWORK_INCLUDE_DIR is ${QT_FRAMEWORK_INCLUDE_DIR}")

    # QT_INCLUDE_DIR points to the QtCore.framework directory, so we need to adjust this to point
    # to the actual include directory, which has include files for non-framework parts of Qt.
    get_filename_component(QT_INCLUDE_DIR "${QT_INCLUDE_DIR}/../../include" ABSOLUTE)
endif()

if(MSVC)
    # Qt5: this flag has changed from /Zc:wchar_t- in Qt4.X
    set(CMAKE_CXX_FLAGS "/Zc:wchar_t /GR /EHsc /DNOCOLOR /DWIN32 /D_WINDOWS /D_SCL_SECURE_NO_WARNINGS") # XXX
else()
    if(CMAKE_HOST_UNIX AND NOT CYGWIN)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -fvisibility=hidden -Wno-strict-aliasing")
    endif()
    set(CMAKE_CXX_FLAGS_DEBUG "-g")
    option(ENABLE_GCC_OPTIMIZATION "Enable specific GCC flags to optimization library size and performance. Only available on Release Mode" 0)
    if(ENABLE_GCC_OPTIMIZATION)
        set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG -Os -Wl,-O1")
        if(NOT CMAKE_HOST_APPLE)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wl,--hash-style=gnu")
        endif()
    endif()

    if(CMAKE_HOST_APPLE)
        # ALTERNATIVE_QT_INCLUDE_DIR is deprecated, because CMake takes care of finding the proper
        # include folders using the qmake found in the environment. Only use it for now in case
        # something goes wrong with the cmake process.
        if(ALTERNATIVE_QT_INCLUDE_DIR AND NOT QT_INCLUDE_DIR)
            set(QT_INCLUDE_DIR ${ALTERNATIVE_QT_INCLUDE_DIR})
        endif()
    endif()
endif()
message(STATUS "*** computed QT_INCLUDE_DIR as ${QT_INCLUDE_DIR}")

set(BINDING_NAME PySide2)

set(BINDING_API_VERSION "${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}.${BINDING_API_MICRO_VERSION}" CACHE STRING "PySide2 version" FORCE)
set(PYSIDE_SO_VERSION ${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION})
if (BINDING_API_PRE_RELEASE_VERSION_TYPE STREQUAL "")
    set(BINDING_API_VERSION_FULL "${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}.${BINDING_API_MICRO_VERSION}"
        CACHE STRING "PySide2 version [full]" FORCE)
else()
    set(BINDING_API_VERSION_FULL "${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}.${BINDING_API_MICRO_VERSION}~${BINDING_API_PRE_RELEASE_VERSION_TYPE}${BINDING_API_PRE_RELEASE_VERSION}"
        CACHE STRING "PySide2 version [full]" FORCE)
endif()

compute_config_py_values(BINDING_API_VERSION)

include(PySideModules)

macro(COLLECT_MODULE_IF_FOUND shortname)
    set(name "Qt5${shortname}")
    set(_qt_module_name "${name}")
    if ("${shortname}" STREQUAL "OpenGLFunctions")
        set(_qt_module_name "Qt5Gui")
    endif()
    # Determine essential/optional/missing
    set(module_state "missing")
    list(FIND ALL_ESSENTIAL_MODULES "${shortname}" essentialIndex)
    if(${essentialIndex} EQUAL -1)
        list(FIND ALL_OPTIONAL_MODULES "${shortname}" optionalIndex)
        if(NOT ${optionalIndex} EQUAL -1)
            set(module_state "optional")
        endif()
    else()
        set(module_state "essential")
    endif()

    # Silence warnings when optional packages are not found when doing a quiet build.
    set(quiet_argument "")
    if (QUIET_BUILD AND "${module_state}" STREQUAL "optional")
        set(quiet_argument "QUIET")
    endif()

    find_package(${_qt_module_name} ${quiet_argument})
    # If package is found, _name_found will be equal to 1
    set(_name_found "${_qt_module_name}_FOUND")
    # _name_dir will keep the path to the directory where the CMake rules were found
    # e.g: ~/qt5.9-install/qtbase/lib/cmake/Qt5Core or /usr/lib64/cmake/Qt5Core
    set(_name_dir "${_qt_module_name}_DIR")
    # Qt5Core will set the base path to check if all the modules are on the same
    # directory, to avoid CMake looking in another path.
    # This will be saved in a global variable at the beginning of the modules
    # collection process.
    string(FIND "${name}" "Qt5Core" qtcore_found)
    if(("${qtcore_found}" GREATER "0") OR ("${qtcore_found}" EQUAL "0"))
        get_filename_component(_core_abs_dir "${${_name_dir}}/../" ABSOLUTE)
        # Setting the absolute path where the Qt5Core was found
        # e.g: ~/qt5.9-install/qtbase/lib/cmake or /usr/lib64/cmake
        message(STATUS "CORE_ABS_DIR:" ${_core_abs_dir})
    endif()

    # Getting the absolute path for each module where the CMake was found, to
    # compare it with CORE_ABS_DIR and check if they are in the same source directory
    # e.g: ~/qt5.9-install/qtbase/lib/cmake/Qt5Script or /usr/lib64/cmake/Qt5Script
    get_filename_component(_module_dir "${${_name_dir}}" ABSOLUTE)
    string(FIND "${_module_dir}" "${_core_abs_dir}" found_basepath)

    # If the module was found, and also the module path is the same as the
    # Qt5Core base path, we will generate the list with the modules to be installed
    set(looked_in_message ". Looked in: ${${_name_dir}}")
    if("${${_name_found}}" AND (("${found_basepath}" GREATER "0") OR ("${found_basepath}" EQUAL "0")))
        message(STATUS "${module_state} module ${name} found (${ARGN})${looked_in_message}")
        # record the shortnames for the tests
        list(APPEND all_module_shortnames ${shortname})
    else()
        if("${module_state}" STREQUAL "optional")
            message(STATUS "optional module ${name} skipped${looked_in_message}")
        elseif("${module_state}" STREQUAL "essential")
            message(STATUS "skipped module ${name} is essential!\n"
                           "   We do not guarantee that all tests are working.${looked_in_message}")
        else()
            message(FATAL_ERROR "module ${name} MISSING${looked_in_message}")
        endif()
    endif()
endmacro()

# Set default values for pyside2_global.h
set (Qt5X11Extras_FOUND "0")
set (Qt5Test_FOUND "0")
set (Qt5Widgets_FOUND "0")

# Collect all essential modules.
# note: the order of this list is relevant for dependencies.
# For instance: Qt5Printsupport must come before Qt5WebKitWidgets.
set(ALL_ESSENTIAL_MODULES Core Gui Widgets PrintSupport Sql Network Test Concurrent)
if(UNIX AND NOT APPLE)
    list(APPEND ALL_ESSENTIAL_MODULES X11Extras)
endif()
if(WIN32)
    list(APPEND ALL_ESSENTIAL_MODULES WinExtras)
endif()
if(APPLE)
    list(APPEND ALL_ESSENTIAL_MODULES MacExtras)
endif()

# Collect all optional modules.
set(ALL_OPTIONAL_MODULES Xml XmlPatterns Help Multimedia
MultimediaWidgets OpenGL OpenGLFunctions Positioning Location Qml Quick QuickWidgets RemoteObjects Scxml Script ScriptTools Sensors TextToSpeech Charts Svg DataVisualization)
find_package(Qt5UiTools)
if(Qt5UiTools_FOUND)
    list(APPEND ALL_OPTIONAL_MODULES UiTools)
else()
    set(DISABLE_QtUiTools 1)
endif()
if(WIN32)
    list(APPEND ALL_OPTIONAL_MODULES AxContainer)
endif()
# Disabling WebKit by default
#   If WebKit support is needed add the following elements
#   to the list: WebKit WebKitWidgets
list(APPEND ALL_OPTIONAL_MODULES WebChannel WebEngineCore WebEngine WebEngineWidgets WebSockets)
if (Qt5Core_VERSION VERSION_GREATER 5.9.3) # Depending on fixes in Qt3D
    list(APPEND ALL_OPTIONAL_MODULES 3DCore 3DRender 3DInput 3DLogic 3DAnimation 3DExtras)
endif()

# Modules to be built unless specified by -DMODULES on command line
if (NOT MODULES)
    set(MODULES "${ALL_ESSENTIAL_MODULES};${ALL_OPTIONAL_MODULES}")
endif()

# This will contain the set of modules for which bindings are not built.
set(DISABLED_MODULES "${ALL_ESSENTIAL_MODULES};${ALL_OPTIONAL_MODULES}")

# Removing from the MODULES list the items that were defined with
# -DSKIP_MODULES on command line
if (SKIP_MODULES)
    foreach(s ${SKIP_MODULES})
        list(REMOVE_ITEM MODULES ${s})
    endforeach()
endif()

foreach(m ${MODULES})
    COLLECT_MODULE_IF_FOUND(${m})
    list(FIND all_module_shortnames ${m} is_module_collected)
    # If the module was collected, remove it from disabled modules list.
    if (NOT is_module_collected EQUAL -1)
        list(REMOVE_ITEM DISABLED_MODULES ${m})
    endif()
endforeach()

# Mark all non-collected modules as disabled. This is used for disabling tests
# that depend on the disabled modules.
foreach(m ${DISABLED_MODULES})
    set(DISABLE_Qt${m} 1)
endforeach()


string(REGEX MATCHALL "[0-9]+" qt_version_helper "${Qt5Core_VERSION}")

list(GET qt_version_helper 0 QT_VERSION_MAJOR)
list(GET qt_version_helper 1 QT_VERSION_MINOR)
unset(qt_version_helper)

set(PYSIDE_QT_VERSION "${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}" CACHE STRING "Qt version used to compile PySide" FORCE)
if(ENABLE_VERSION_SUFFIX)
      set(pyside_SUFFIX "-${BINDING_API_MAJOR_VERSION}.${BINDING_API_MINOR_VERSION}")
endif()

# no more supported: include(${QT_USE_FILE})

# Configure OS support
set(ENABLE_X11 "0")
set(ENABLE_MAC "0")
set(ENABLE_WIN "0")
set(ENABLE_SIMULATOR "0")

# no more Maemo, no more simulator
if(CMAKE_HOST_APPLE)
    set(ENABLE_MAC "1")
    set(AUTO_OS "mac")
elseif(CMAKE_HOST_WIN32)
    set(ENABLE_WIN "1")
    set(AUTO_OS "win")
elseif(CMAKE_HOST_UNIX)
    set(ENABLE_X11 "1")
    set(AUTO_OS "x11")
else()
    message(FATAL_ERROR "OS not supported")
endif()
message(STATUS "Detected OS: ${AUTO_OS}")

# Define supported Qt Version
set(SUPPORTED_QT_VERSION "${QT_VERSION_MAJOR}.${QT_VERSION_MINOR}")


# uninstall target
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake"
               "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
               IMMEDIATE @ONLY)

# When opening super project, prevent redefinition of uninstall target.
if (NOT TARGET uninstall)
    add_custom_target(uninstall "${CMAKE_COMMAND}"
                      -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
endif()

if (NOT PYTHON_SITE_PACKAGES)
    execute_process(
        COMMAND ${SHIBOKEN_PYTHON_INTERPRETER} -c "if True:
            from distutils import sysconfig
            from os.path import sep
            print(sysconfig.get_python_lib(1, 0, prefix='${CMAKE_INSTALL_PREFIX}').replace(sep, '/'))
            "
        OUTPUT_VARIABLE PYTHON_SITE_PACKAGES
        OUTPUT_STRIP_TRAILING_WHITESPACE)
    if (NOT PYTHON_SITE_PACKAGES)
        message(FATAL_ERROR "Could not detect Python module installation directory.")
    elseif (APPLE)
        message(STATUS "!!! The generated bindings will be installed on ${PYTHON_SITE_PACKAGES}, is it right!?")
    endif()
endif()

set(GENERATOR_EXTRA_FLAGS --generator-set=shiboken
                          --enable-parent-ctor-heuristic
                          --enable-pyside-extensions
                          --enable-return-value-heuristic
                          --use-isnull-as-nb_nonzero)
# 2017-04-24 The protected hack can unfortunately not be disabled, because
# Clang does produce linker errors when we disable the hack.
# But the ugly workaround in Python is replaced by a shiboken change.
if(WIN32 OR DEFINED AVOID_PROTECTED_HACK)
    message(STATUS "PySide2 will be generated avoiding the protected hack!")
    set(GENERATOR_EXTRA_FLAGS ${GENERATOR_EXTRA_FLAGS} --avoid-protected-hack)
    add_definitions(-DAVOID_PROTECTED_HACK)
else()
    message(STATUS "PySide will be generated using the protected hack!")
endif()

# Build with Address sanitizer enabled if requested. This may break things, so use at your own risk.
if (SANITIZE_ADDRESS AND NOT MSVC)
    # Currently this does not check that the clang / gcc version used supports Address sanitizer,
    # so once again, use at your own risk.
    add_compile_options("-fsanitize=address" "-g" "-fno-omit-frame-pointer")
    # We need to add the sanitize address option to all linked executables / shared libraries
    # so that proper sanitizer symbols are linked in.
    #
    # Note that when running tests, you may need to set an additional environment variable
    # in set_tests_properties for shiboken2 / pyside tests, or exported in your shell. Address
    # sanitizer will tell you what environment variable needs to be exported. For example:
    # export DYLD_INSERT_LIBRARIES=/Applications/Xcode.app/Contents/Developer/Toolchains/
    #   ./XcodeDefault.xctoolchain/usr/lib/clang/8.1.0/lib/darwin/libclang_rt.asan_osx_dynamic.dylib
    set(CMAKE_CXX_STANDARD_LIBRARIES "${CMAKE_STANDARD_LIBRARIES} -fsanitize=address")
endif()

add_subdirectory(libpyside)
find_package(Qt5Designer)
if(Qt5UiTools_FOUND AND Qt5Designer_FOUND)
    add_subdirectory(plugins)
endif()

# project directories
add_subdirectory(PySide2)
if (BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif ()

find_program(SPHINX_BUILD sphinx-build)
find_program(DOT_EXEC dot)

if (QT_SRC_DIR AND SPHINX_BUILD AND DOT_EXEC)
    add_subdirectory(doc)
else ()
    set(DOCS_TARGET_DISABLED_MESSAGE "apidoc generation targets disabled.")
    if (NOT QT_SRC_DIR)
        message(STATUS "QT_SRC_DIR variable not set, ${DOCS_TARGET_DISABLED_MESSAGE}")
    elseif (NOT SPHINX_BUILD)
        message(STATUS "sphinx-build command not found, ${DOCS_TARGET_DISABLED_MESSAGE}")
    elseif (NOT DOT_EXEC)
        message(STATUS "graphviz not found, ${DOCS_TARGET_DISABLED_MESSAGE}")
    else()
        message(STATUS "Unknown issue occurred, ${DOCS_TARGET_DISABLED_MESSAGE}")
    endif()
endif()
