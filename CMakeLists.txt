set(CMAKE_LEGACY_CYGWIN_WIN32 OFF)

cmake_minimum_required(VERSION 2.8.12)

project(glfw)

find_package(catkin REQUIRED)

catkin_package(
  INCLUDE_DIRS include
  LIBRARIES glfw
  CATKIN_DEPENDS
  DEPENDS
)

if (NOT CMAKE_VERSION VERSION_LESS "3.0")
    # Until all major package systems have moved to CMake 3,
    # we stick with the older INSTALL_NAME_DIR mechanism
    cmake_policy(SET CMP0042 OLD)
endif()

set(GLFW_VERSION_MAJOR "3")
set(GLFW_VERSION_MINOR "2")
set(GLFW_VERSION_PATCH "1")
set(GLFW_VERSION_EXTRA "")
set(GLFW_VERSION "${GLFW_VERSION_MAJOR}.${GLFW_VERSION_MINOR}")
set(GLFW_VERSION_FULL "${GLFW_VERSION}.${GLFW_VERSION_PATCH}${GLFW_VERSION_EXTRA}")
set(LIB_SUFFIX "" CACHE STRING "Takes an empty string or 64. Directory where lib will be installed: lib or lib64")

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(GLFW_BUILD_EXAMPLES "Build the GLFW example programs" OFF)
option(GLFW_BUILD_TESTS "Build the GLFW test programs" OFF)
option(GLFW_BUILD_DOCS "Build the GLFW documentation" OFF)
option(GLFW_INSTALL "Generate installation target" OFF)
option(GLFW_VULKAN_STATIC "Use the Vulkan loader statically linked into application" OFF)
option(GLFW_DOCUMENT_INTERNALS "Include internals in documentation" OFF)

if (WIN32)
    option(GLFW_USE_HYBRID_HPG "Force use of high-performance GPU on hybrid systems" OFF)
endif()

if (APPLE)
    option(GLFW_USE_CHDIR "Make glfwInit chdir to Contents/Resources" ON)
    option(GLFW_USE_MENUBAR "Populate the menu bar on first window creation" ON)
    option(GLFW_USE_RETINA "Use the full resolution of Retina displays" ON)
endif()

if (UNIX AND NOT APPLE)
    option(GLFW_USE_WAYLAND "Use Wayland for window creation" OFF)
    option(GLFW_USE_MIR     "Use Mir for window creation" OFF)
endif()

if (MSVC)
    option(USE_MSVC_RUNTIME_LIBRARY_DLL "Use MSVC runtime library DLL" ON)
endif()

if (BUILD_SHARED_LIBS)
    set(_GLFW_BUILD_DLL 1)
endif()

if (BUILD_SHARED_LIBS AND UNIX)
    # On Unix-like systems, shared libraries can use the soname system.
    set(GLFW_LIB_NAME glfw)
else()
    set(GLFW_LIB_NAME glfw3)
endif()

if (GLFW_VULKAN_STATIC)
    set(_GLFW_VULKAN_STATIC 1)
endif()

list(APPEND CMAKE_MODULE_PATH "${GLFW_SOURCE_DIR}/CMake/modules")

find_package(Threads REQUIRED)
find_package(Vulkan)

if (GLFW_BUILD_DOCS)
    set(DOXYGEN_SKIP_DOT TRUE)
    find_package(Doxygen)
endif()

#--------------------------------------------------------------------
# Set compiler specific flags
#--------------------------------------------------------------------
if (MSVC)
    if (NOT USE_MSVC_RUNTIME_LIBRARY_DLL)
        foreach (flag CMAKE_C_FLAGS
                      CMAKE_C_FLAGS_DEBUG
                      CMAKE_C_FLAGS_RELEASE
                      CMAKE_C_FLAGS_MINSIZEREL
                      CMAKE_C_FLAGS_RELWITHDEBINFO)

            if (${flag} MATCHES "/MD")
                string(REGEX REPLACE "/MD" "/MT" ${flag} "${${flag}}")
            endif()
            if (${flag} MATCHES "/MDd")
                string(REGEX REPLACE "/MDd" "/MTd" ${flag} "${${flag}}")
            endif()

        endforeach()
    endif()
endif()

if (MINGW)
    # Workaround for legacy MinGW not providing XInput and DirectInput
    include(CheckIncludeFile)

    check_include_file(dinput.h DINPUT_H_FOUND)
    check_include_file(xinput.h XINPUT_H_FOUND)
    if (NOT DINPUT_H_FOUND OR NOT XINPUT_H_FOUND)
        list(APPEND glfw_INCLUDE_DIRS "${GLFW_SOURCE_DIR}/deps/mingw")
    endif()

    # Enable link-time exploit mitigation features enabled by default on MSVC
    include(CheckCCompilerFlag)

    # Compatibility with data execution prevention (DEP)
    set(CMAKE_REQUIRED_FLAGS "-Wl,--nxcompat")
    check_c_compiler_flag("" _GLFW_HAS_DEP)
    if (_GLFW_HAS_DEP)
        set(CMAKE_SHARED_LINKER_FLAGS "-Wl,--nxcompat ${CMAKE_SHARED_LINKER_FLAGS}")
    endif()

    # Compatibility with address space layout randomization (ASLR)
    set(CMAKE_REQUIRED_FLAGS "-Wl,--dynamicbase")
    check_c_compiler_flag("" _GLFW_HAS_ASLR)
    if (_GLFW_HAS_ASLR)
        set(CMAKE_SHARED_LINKER_FLAGS "-Wl,--dynamicbase ${CMAKE_SHARED_LINKER_FLAGS}")
    endif()

    # Compatibility with 64-bit address space layout randomization (ASLR)
    set(CMAKE_REQUIRED_FLAGS "-Wl,--high-entropy-va")
    check_c_compiler_flag("" _GLFW_HAS_64ASLR)
    if (_GLFW_HAS_64ASLR)
        set(CMAKE_SHARED_LINKER_FLAGS "-Wl,--high-entropy-va ${CMAKE_SHARED_LINKER_FLAGS}")
    endif()
endif()

#--------------------------------------------------------------------
# Detect and select backend APIs
#--------------------------------------------------------------------
if (WIN32)
    set(_GLFW_WIN32 1)
    message(STATUS "Using Win32 for window creation")
elseif (APPLE)
    set(_GLFW_COCOA 1)
    message(STATUS "Using Cocoa for window creation")
elseif (UNIX)
    if (GLFW_USE_WAYLAND)
        set(_GLFW_WAYLAND 1)
        message(STATUS "Using Wayland for window creation")
    elseif (GLFW_USE_MIR)
        set(_GLFW_MIR 1)
        message(STATUS "Using Mir for window creation")
    else()
        set(_GLFW_X11 1)
        message(STATUS "Using X11 for window creation")
    endif()
else()
    message(FATAL_ERROR "No supported platform was detected")
endif()

#--------------------------------------------------------------------
# Add Vulkan static library if requested
#--------------------------------------------------------------------
if (GLFW_VULKAN_STATIC)
    if (VULKAN_FOUND AND VULKAN_STATIC_LIBRARY)
        list(APPEND glfw_LIBRARIES ${VULKAN_STATIC_LIBRARY})
    else()
        if (BUILD_SHARED_LIBS OR GLFW_BUILD_EXAMPLES OR GLFW_BUILD_TESTS)
            message(FATAL_ERROR "Vulkan loader static library not found")
        else()
            message(WARNING "Vulkan loader static library not found")
        endif()
    endif()
endif()

#--------------------------------------------------------------------
# Find and add Unix math and time libraries
#--------------------------------------------------------------------
if (UNIX AND NOT APPLE)
    find_library(RT_LIBRARY rt)
    mark_as_advanced(RT_LIBRARY)
    if (RT_LIBRARY)
        list(APPEND glfw_LIBRARIES "${RT_LIBRARY}")
        list(APPEND glfw_PKG_LIBS "-lrt")
    endif()

    find_library(MATH_LIBRARY m)
    mark_as_advanced(MATH_LIBRARY)
    if (MATH_LIBRARY)
        list(APPEND glfw_LIBRARIES "${MATH_LIBRARY}")
        list(APPEND glfw_PKG_LIBS "-lm")
    endif()

    if (CMAKE_DL_LIBS)
        list(APPEND glfw_LIBRARIES "${CMAKE_DL_LIBS}")
        list(APPEND glfw_PKG_LIBS "-l${CMAKE_DL_LIBS}")
    endif()
endif()

#--------------------------------------------------------------------
# Use Win32 for window creation
#--------------------------------------------------------------------
if (_GLFW_WIN32)

    list(APPEND glfw_PKG_LIBS "-lgdi32")

    if (GLFW_USE_HYBRID_HPG)
        set(_GLFW_USE_HYBRID_HPG 1)
    endif()
endif()

#--------------------------------------------------------------------
# Use X11 for window creation
#--------------------------------------------------------------------
if (_GLFW_X11)

    find_package(X11 REQUIRED)

    list(APPEND glfw_PKG_DEPS "x11")

    # Set up library and include paths
    list(APPEND glfw_INCLUDE_DIRS "${X11_X11_INCLUDE_PATH}")
    list(APPEND glfw_LIBRARIES "${X11_X11_LIB}" "${CMAKE_THREAD_LIBS_INIT}")

    # Check for XRandR (modern resolution switching and gamma control)
    if (NOT X11_Xrandr_FOUND)
        message(FATAL_ERROR "The RandR library and headers were not found")
    endif()

    list(APPEND glfw_INCLUDE_DIRS "${X11_Xrandr_INCLUDE_PATH}")
    list(APPEND glfw_LIBRARIES "${X11_Xrandr_LIB}")
    list(APPEND glfw_PKG_DEPS "xrandr")

    # Check for Xinerama (legacy multi-monitor support)
    if (NOT X11_Xinerama_FOUND)
        message(FATAL_ERROR "The Xinerama library and headers were not found")
    endif()

    list(APPEND glfw_INCLUDE_DIRS "${X11_Xinerama_INCLUDE_PATH}")
    list(APPEND glfw_LIBRARIES "${X11_Xinerama_LIB}")
    list(APPEND glfw_PKG_DEPS "xinerama")

    # Check for Xf86VidMode (fallback gamma control)
    if (X11_xf86vmode_FOUND)
        list(APPEND glfw_INCLUDE_DIRS "${X11_xf86vmode_INCLUDE_PATH}")
        list(APPEND glfw_PKG_DEPS "xxf86vm")

        if (X11_Xxf86vm_LIB)
            list(APPEND glfw_LIBRARIES "${X11_Xxf86vm_LIB}")
        else()
            # Backwards compatibility (see CMake bug 0006976)
            list(APPEND glfw_LIBRARIES Xxf86vm)
        endif()

        set(_GLFW_HAS_XF86VM TRUE)
    endif()

    # Check for Xkb (X keyboard extension)
    if (NOT X11_Xkb_FOUND)
        message(FATAL_ERROR "The X keyboard extension headers were not found")
    endif()

    list(APPEND glfw_INCLUDE_DIR "${X11_Xkb_INCLUDE_PATH}")

    # Check for Xcursor
    if (NOT X11_Xcursor_FOUND)
        message(FATAL_ERROR "The Xcursor libraries and headers were not found")
    endif()

    list(APPEND glfw_INCLUDE_DIR "${X11_Xcursor_INCLUDE_PATH}")
    list(APPEND glfw_LIBRARIES "${X11_Xcursor_LIB}")
    list(APPEND glfw_PKG_DEPS "xcursor")

endif()

#--------------------------------------------------------------------
# Use Wayland for window creation
#--------------------------------------------------------------------
if (_GLFW_WAYLAND)
    find_package(ECM REQUIRED NO_MODULE)
    list(APPEND CMAKE_MODULE_PATH ${ECM_MODULE_PATH})

    find_package(Wayland REQUIRED)
    find_package(WaylandScanner REQUIRED)
    find_package(WaylandProtocols 1.1 REQUIRED)

    list(APPEND glfw_PKG_DEPS "wayland-egl")

    list(APPEND glfw_INCLUDE_DIRS "${Wayland_INCLUDE_DIR}")
    list(APPEND glfw_LIBRARIES "${Wayland_LIBRARIES}" "${CMAKE_THREAD_LIBS_INIT}")

    find_package(XKBCommon REQUIRED)
    list(APPEND glfw_PKG_DEPS "xkbcommon")
    list(APPEND glfw_INCLUDE_DIRS "${XKBCOMMON_INCLUDE_DIRS}")
    list(APPEND glfw_LIBRARIES "${XKBCOMMON_LIBRARY}")
endif()

#--------------------------------------------------------------------
# Use Mir for window creation
#--------------------------------------------------------------------
if (_GLFW_MIR)
    find_package(Mir REQUIRED)
    list(APPEND glfw_PKG_DEPS "mirclient")

    list(APPEND glfw_INCLUDE_DIRS "${MIR_INCLUDE_DIR}")
    list(APPEND glfw_LIBRARIES "${MIR_LIBRARIES}" "${CMAKE_THREAD_LIBS_INIT}")

    find_package(XKBCommon REQUIRED)
    list(APPEND glfw_PKG_DEPS "xkbcommon")
    list(APPEND glfw_INCLUDE_DIRS "${XKBCOMMON_INCLUDE_DIRS}")
    list(APPEND glfw_LIBRARIES "${XKBCOMMON_LIBRARY}")
endif()

#--------------------------------------------------------------------
# Use Cocoa for window creation and NSOpenGL for context creation
#--------------------------------------------------------------------
if (_GLFW_COCOA)

    if (GLFW_USE_MENUBAR)
        set(_GLFW_USE_MENUBAR 1)
    endif()

    if (GLFW_USE_CHDIR)
        set(_GLFW_USE_CHDIR 1)
    endif()

    if (GLFW_USE_RETINA)
        set(_GLFW_USE_RETINA 1)
    endif()

    # Set up library and include paths
    find_library(COCOA_FRAMEWORK Cocoa)
    find_library(IOKIT_FRAMEWORK IOKit)
    find_library(CORE_FOUNDATION_FRAMEWORK CoreFoundation)
    find_library(CORE_VIDEO_FRAMEWORK CoreVideo)
    mark_as_advanced(COCOA_FRAMEWORK
                     IOKIT_FRAMEWORK
                     CORE_FOUNDATION_FRAMEWORK
                     CORE_VIDEO_FRAMEWORK)
    list(APPEND glfw_LIBRARIES "${COCOA_FRAMEWORK}"
                               "${IOKIT_FRAMEWORK}"
                               "${CORE_FOUNDATION_FRAMEWORK}"
                               "${CORE_VIDEO_FRAMEWORK}")

    set(glfw_PKG_DEPS "")
    set(glfw_PKG_LIBS "-framework Cocoa -framework IOKit -framework CoreFoundation -framework CoreVideo")
endif()

#--------------------------------------------------------------------
# Export GLFW library dependencies
#--------------------------------------------------------------------
foreach(arg ${glfw_PKG_DEPS})
    set(GLFW_PKG_DEPS "${GLFW_PKG_DEPS} ${arg}")
endforeach()
foreach(arg ${glfw_PKG_LIBS})
    set(GLFW_PKG_LIBS "${GLFW_PKG_LIBS} ${arg}")
endforeach()

#--------------------------------------------------------------------
# Create generated files
#--------------------------------------------------------------------
#include(CMakePackageConfigHelpers)

#set(GLFW_CONFIG_PATH "lib${LIB_SUFFIX}/cmake/glfw3")

#configure_package_config_file(src/glfw3Config.cmake.in
#                              src/glfw3Config.cmake
#                              INSTALL_DESTINATION "${GLFW_CONFIG_PATH}"
#                              NO_CHECK_REQUIRED_COMPONENTS_MACRO)

#write_basic_package_version_file(src/glfw3ConfigVersion.cmake
#                                 VERSION ${GLFW_VERSION_FULL}
#                                 COMPATIBILITY SameMajorVersion)

#configure_file(src/glfw_config.h.in src/glfw_config.h @ONLY)

#configure_file(src/glfw3.pc.in src/glfw3.pc @ONLY)

#--------------------------------------------------------------------
# Add subdirectories
#--------------------------------------------------------------------
set(common_HEADERS src/internal.h
                   src/glfw_config.h
                   include/glfw/glfw3.h
                   include/glfw/glfw3native.h)
set(common_SOURCES src/context.c src/init.c src/input.c src/monitor.c src/vulkan.c src/window.c)

if (_GLFW_COCOA)
    set(glfw_HEADERS ${common_HEADERS} src/cocoa_platform.h src/cocoa_joystick.h
                     src/posix_tls.h src/nsgl_context.h)
    set(glfw_SOURCES ${common_SOURCES} src/cocoa_init.m src/cocoa_joystick.m
                     src/cocoa_monitor.m src/cocoa_window.m src/cocoa_time.c src/posix_tls.c
                     src/nsgl_context.m)
elseif (_GLFW_WIN32)
    set(glfw_HEADERS ${common_HEADERS} src/win32_platform.h src/win32_joystick.h
                     src/wgl_context.h src/egl_context.h)
    set(glfw_SOURCES ${common_SOURCES} src/win32_init.c src/win32_joystick.c
                     src/win32_monitor.c src/win32_time.c src/win32_tls.c src/win32_window.c
                     src/wgl_context.c src/egl_context.c)
elseif (_GLFW_X11)
    set(glfw_HEADERS ${common_HEADERS} src/x11_platform.h src/xkb_unicode.h
                     src/linux_joystick.h src/posix_time.h src/posix_tls.h src/glx_context.h
                     src/egl_context.h)
    set(glfw_SOURCES ${common_SOURCES} src/x11_init.c src/x11_monitor.c src/x11_window.c
                     src/xkb_unicode.c src/linux_joystick.c src/posix_time.c src/posix_tls.c
                     src/glx_context.c src/egl_context.c)
elseif (_GLFW_WAYLAND)
    set(glfw_HEADERS ${common_HEADERS} src/wl_platform.h src/linux_joystick.h
                     src/posix_time.h src/posix_tls.h src/xkb_unicode.h src/egl_context.h)
    set(glfw_SOURCES ${common_SOURCES} src/wl_init.c src/wl_monitor.c src/wl_window.c
                     src/linux_joystick.c src/posix_time.c src/posix_tls.c src/xkb_unicode.c
                     src/egl_context.c)

    ecm_add_wayland_client_protocol(glfw_SOURCES
        PROTOCOL
        ${WAYLAND_PROTOCOLS_PKGDATADIR}/unstable/relative-pointer/relative-pointer-unstable-v1.xml
        BASENAME relative-pointer-unstable-v1)
    ecm_add_wayland_client_protocol(glfw_SOURCES
        PROTOCOL
        ${WAYLAND_PROTOCOLS_PKGDATADIR}/unstable/pointer-constraints/pointer-constraints-unstable-v1.xml
        BASENAME pointer-constraints-unstable-v1)
elseif (_GLFW_MIR)
    set(glfw_HEADERS ${common_HEADERS} src/mir_platform.h src/linux_joystick.h
                     src/posix_time.h src/posix_tls.h src/xkb_unicode.h src/egl_context.h)
    set(glfw_SOURCES ${common_SOURCES} src/mir_init.c src/mir_monitor.c src/mir_window.c
                     src/linux_joystick.c src/posix_time.c src/posix_tls.c src/xkb_unicode.c
                     src/egl_context.c)
endif()

if (APPLE)
    # For some reason, CMake doesn't know about .m
    set_source_files_properties(${glfw_SOURCES} PROPERTIES LANGUAGE C)
endif()

add_library(glfw ${glfw_SOURCES} ${glfw_HEADERS})
set_target_properties(glfw PROPERTIES
                      OUTPUT_NAME ${GLFW_LIB_NAME}
                      VERSION ${GLFW_VERSION}
                      SOVERSION ${GLFW_VERSION_MAJOR}
                      POSITION_INDEPENDENT_CODE ON
                      FOLDER "GLFW3")

target_compile_definitions(glfw PRIVATE -D_GLFW_USE_CONFIG_H)
target_include_directories(glfw PUBLIC
                           $<BUILD_INTERFACE:${GLFW_SOURCE_DIR}/include>
                           $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/include>)
target_include_directories(glfw PRIVATE
                           "${GLFW_SOURCE_DIR}/src"
                           "${GLFW_BINARY_DIR}/src"
                           ${glfw_INCLUDE_DIRS})

# HACK: When building on MinGW, WINVER and UNICODE need to be defined before
# the inclusion of stddef.h (by glfw3.h), which is itself included before
# win32_platform.h.  We define them here until a saner solution can be found
# NOTE: MinGW-w64 and Visual C++ do /not/ need this hack.
target_compile_definitions(glfw PRIVATE
                           "$<$<BOOL:${MINGW}>:UNICODE;WINVER=0x0501>")

# Enable a reasonable set of warnings (no, -Wextra is not reasonable)
target_compile_options(glfw PRIVATE
                       "$<$<C_COMPILER_ID:Clang>:-Wall>"
                       "$<$<C_COMPILER_ID:GNU>:-Wall>")

if (BUILD_SHARED_LIBS)
    if (WIN32)
        if (MINGW)
            # Remove the lib prefix on the DLL (but not the import library
            set_target_properties(glfw PROPERTIES PREFIX "")

            # Add a suffix to the import library to avoid naming conflicts
            set_target_properties(glfw PROPERTIES IMPORT_SUFFIX "dll.a")
        else()
            # Add a suffix to the import library to avoid naming conflicts
            set_target_properties(glfw PROPERTIES IMPORT_SUFFIX "dll.lib")
        endif()
    elseif (APPLE)
        # Add -fno-common to work around a bug in Apple's GCC
        target_compile_options(glfw PRIVATE "-fno-common")

        set_target_properties(glfw PROPERTIES
                              INSTALL_NAME_DIR "lib${LIB_SUFFIX}")
    elseif (UNIX)
        # Hide symbols not explicitly tagged for export from the shared library
        target_compile_options(glfw PRIVATE "-fvisibility=hidden")
    endif()

    target_compile_definitions(glfw INTERFACE -DGLFW_DLL)
    target_link_libraries(glfw PRIVATE ${glfw_LIBRARIES})
else()
    target_link_libraries(glfw INTERFACE ${glfw_LIBRARIES})
endif()

if (MSVC)
    target_compile_definitions(glfw PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

#if (GLFW_INSTALL)
#    install(TARGETS glfw EXPORT glfwTargets DESTINATION lib${LIB_SUFFIX})
#endif()

install(TARGETS glfw
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

if (GLFW_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if (GLFW_BUILD_TESTS)
    add_subdirectory(tests)
endif()

if (DOXYGEN_FOUND AND GLFW_BUILD_DOCS)
    add_subdirectory(docs)
endif()

#--------------------------------------------------------------------
# Install files other than the library
#--------------------------------------------------------------------
install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  PATTERN ".svn" EXCLUDE
)
