cmake_minimum_required(VERSION 3.14)
project(sam3.cpp VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# ─────────────────────────────────────────────────────────────────────────
# External ggml option
#
# When OFF (default): vendor ggml as a subdirectory — the upstream behavior.
# When ON : skip the bundled ggml and resolve `find_package(ggml CONFIG)`
#           against an installed ggml (BUILD_SHARED_LIBS=ON installs a
#           ggmlConfig.cmake + ggml-*.lib import libs). Lets a downstream
#           project link multiple ggml-based libraries (sam3 + llama.cpp +
#           stable-diffusion.cpp) against a single shared ggml so they don't
#           ship duplicate copies into the same process.
# ─────────────────────────────────────────────────────────────────────────
option(SAM3_USE_EXTERNAL_GGML "Consume external ggml via find_package() instead of the bundled submodule" OFF)

# ggml options — enable Metal on Apple (bundled-ggml path only)
option(SAM3_METAL "Enable Metal backend" ON)
if(APPLE AND SAM3_METAL AND NOT SAM3_USE_EXTERNAL_GGML)
    set(GGML_METAL        ON CACHE BOOL "" FORCE)
    set(GGML_METAL_EMBED_LIBRARY ON CACHE BOOL "" FORCE)
endif()

if(SAM3_USE_EXTERNAL_GGML)
    # find_package picks up the ggmlConfig.cmake from the install prefix the
    # caller pointed `ggml_DIR` (or CMAKE_PREFIX_PATH) at. Provides the `ggml`
    # IMPORTED target with includes + the appropriate import lib.
    find_package(ggml CONFIG REQUIRED)
else()
    add_subdirectory(ggml)
endif()

# sam3 library — honor BUILD_SHARED_LIBS so the downstream CMake can ask for
# a sam3.dll/.so without patching this file. When shared, dllexport every
# public symbol via SAM3_API (see sam3.h).
add_library(sam3 sam3.cpp sam3.h)
target_include_directories(sam3 PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/stb
)
target_link_libraries(sam3 PUBLIC ggml)
target_compile_features(sam3 PUBLIC cxx_std_14)

# Public-visibility position-independent code so static libs can still be
# linked into a downstream shared lib.
set_property(TARGET sam3 PROPERTY POSITION_INDEPENDENT_CODE ON)

if(BUILD_SHARED_LIBS)
    # SAM3_SHARED is public so both the sam3 TU and the consumers see the
    # macro (and pick dllexport vs dllimport correctly). SAM3_BUILD is private
    # so only sam3.cpp itself emits dllexport; consumers see dllimport.
    target_compile_definitions(sam3 PUBLIC  SAM3_SHARED)
    target_compile_definitions(sam3 PRIVATE SAM3_BUILD)
endif()

# Examples (only when top-level)
option(SAM3_BUILD_EXAMPLES "Build example executables" ON)
if(SAM3_BUILD_EXAMPLES AND CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    add_subdirectory(examples)
endif()

# Tests
option(SAM3_BUILD_TESTS "Build test executables" OFF)
if(SAM3_BUILD_TESTS)
    add_subdirectory(tests)
endif()
