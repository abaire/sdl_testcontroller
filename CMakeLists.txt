cmake_minimum_required(VERSION 3.16)
project(testcontroller C)

if(NOT DEFINED CMAKE_C_STANDARD)
    set(CMAKE_C_STANDARD 99)
endif()

# ---------------------------------------------------------------------------
# Locate SDL3
# ---------------------------------------------------------------------------
# First try to find an already-installed SDL3.  If not found, fall back to
# FetchContent so the project works without any prior setup.
find_package(SDL3 QUIET)

if(NOT SDL3_FOUND)
    message(STATUS "SDL3 not found via find_package – fetching from GitHub")
    include(FetchContent)
    FetchContent_Declare(
        SDL3
        GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
        GIT_TAG        main
        GIT_SHALLOW    ON
    )
    # Build SDL3 as a shared library (mirrors the default test setup)
    set(SDL_SHARED ON  CACHE BOOL "" FORCE)
    set(SDL_STATIC OFF CACHE BOOL "" FORCE)
    set(SDL_TEST_LIBRARY ON CACHE BOOL "" FORCE)
    FetchContent_MakeAvailable(SDL3)
endif()

# ---------------------------------------------------------------------------
# Resolve the correct link targets depending on how SDL3 was found
# ---------------------------------------------------------------------------
# When SDL3 is found via find_package, targets are SDL3::SDL3 and SDL3::SDL3_test.
# When built via FetchContent the same target names are created by SDL's own
# CMakeLists, so the names are consistent either way.

if(TARGET SDL3::SDL3)
    set(SDL3_LIB SDL3::SDL3)
elseif(TARGET SDL3-shared)
    set(SDL3_LIB SDL3-shared)
else()
    set(SDL3_LIB SDL3::SDL3-static)
endif()

if(TARGET SDL3::SDL3_test)
    set(SDL3_TEST_LIB SDL3::SDL3_test)
elseif(TARGET SDL3_test)
    set(SDL3_TEST_LIB SDL3_test)
else()
    message(FATAL_ERROR "Could not locate the SDL3 test library target. "
        "Make sure SDL3 is built with SDL_TEST_LIBRARY=ON.")
endif()

# ---------------------------------------------------------------------------
# testcontroller executable
# ---------------------------------------------------------------------------
add_executable(testcontroller
    src/testcontroller.c
    src/gamepadutils.c
    src/testutils.c
)

target_include_directories(testcontroller PRIVATE
    src   # so #include "gamepadutils.h" etc. resolve
)

target_link_libraries(testcontroller PRIVATE
    ${SDL3_TEST_LIB}
    ${SDL3_LIB}
)

# SDL_MAIN_USE_CALLBACKS replaces main(); SDL_main.h provides the platform
# entry point.  On Apple platforms we still need an app-bundle wrapper so
# SDL_main.h can do its work.
if(APPLE)
    set_target_properties(testcontroller PROPERTIES
        MACOSX_BUNDLE TRUE
        MACOSX_BUNDLE_GUI_IDENTIFIER "org.libsdl.testcontroller"
        MACOSX_BUNDLE_BUNDLE_VERSION "1.0"
        MACOSX_BUNDLE_SHORT_VERSION_STRING "1.0"
    )
endif()

# ---------------------------------------------------------------------------
# Install
# ---------------------------------------------------------------------------
install(TARGETS testcontroller
    RUNTIME DESTINATION bin
    BUNDLE  DESTINATION .
)
