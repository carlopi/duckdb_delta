cmake_minimum_required(VERSION 2.8.12)
include(ExternalProject)

### Core config
set(TARGET_NAME delta)

set(EXTENSION_NAME ${TARGET_NAME}_extension)
set(LOADABLE_EXTENSION_NAME ${TARGET_NAME}_loadable_extension)

project(${TARGET_NAME})

include_directories(src/include)

set(EXTENSION_SOURCES
        src/delta_extension.cpp
        src/delta_functions.cpp
        src/delta_utils.cpp
        src/functions/delta_scan.cpp)

### Setup delta-kernel-rs dependency
set(KERNEL_NAME delta_kernel)

# Set default ExternalProject root directory
set_directory_properties(PROPERTIES EP_PREFIX ${CMAKE_BINARY_DIR}/rust)

set(RUST_ENV_VARS "")

# Propagate arch to rust build for CI
set(RUST_PLATFORM_TARGET "wasm32-unknown-emscripten")

# We currently only support the predefined targets.
if ("${RUST_PLATFORM_TARGET}" STREQUAL "")
    message(FATAL_ERROR "Failed to detect the correct platform")
endif()

set(RUST_PLATFORM_PARAM "--target=${RUST_PLATFORM_TARGET}")
message(STATUS "Building for rust target: ${RUST_PLATFORM_TARGET}")

# Remove whitespaces before and after to prevent messed up env variables
string(STRIP "${RUST_ENV_VARS}" RUST_ENV_VARS)

# Having these set will mess up cross compilation to linux arm
set(RUST_UNSET_ENV_VARS --unset=CC --unset=CXX --unset=LD)

# Define all the relevant delta-kernel-rs paths/names
set(DELTA_KERNEL_LIBNAME "${CMAKE_STATIC_LIBRARY_PREFIX}delta_kernel_ffi${CMAKE_STATIC_LIBRARY_SUFFIX}")
set(DELTA_KERNEL_LIBPATH_DEBUG "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/debug/${DELTA_KERNEL_LIBNAME}")
set(DELTA_KERNEL_LIBPATH_RELEASE "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/release/${DELTA_KERNEL_LIBNAME}")
set(DELTA_KERNEL_FFI_HEADER_PATH "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/ffi-headers")
set(DELTA_KERNEL_FFI_HEADER_C "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/ffi-headers/delta_kernel_ffi.h")
set(DELTA_KERNEL_FFI_HEADER_CXX "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/ffi-headers/delta_kernel_ffi.hpp")

# Add rust_example as a CMake target
ExternalProject_Add(
        ${KERNEL_NAME}
        GIT_REPOSITORY "https://github.com/carlopi/delta-kernel-rs"
        # WARNING: the FFI headers are currently pinned due to the C linkage issue of the c++ headers. Currently, when bumping
        # the kernel version, the produced header in ./src/include/delta_kernel_ffi.hpp should be also bumped, applying the fix
        GIT_TAG experiment_wasm
        # Prints the env variables passed to the cargo build to the terminal, useful in debugging because passing them
        # through CMake is an error-prone mess
        CONFIGURE_COMMAND ${CMAKE_COMMAND} -E env ${RUST_UNSET_ENV_VARS} ${RUST_ENV_VARS} env
        UPDATE_COMMAND ""
        BUILD_IN_SOURCE 1
        # Build debug build
        BUILD_COMMAND
            ${CMAKE_COMMAND} -E env ${RUST_UNSET_ENV_VARS} ${RUST_ENV_VARS}
            cargo build --package delta_kernel_ffi --features sync-engine  ${RUST_PLATFORM_PARAM}
        # Define the byproducts, required for building with Ninja
        BUILD_BYPRODUCTS "${DELTA_KERNEL_LIBPATH_DEBUG}"
        BUILD_BYPRODUCTS "${DELTA_KERNEL_LIBPATH_RELEASE}"
        BUILD_BYPRODUCTS "${DELTA_KERNEL_FFI_HEADER_C}"
        BUILD_BYPRODUCTS "${DELTA_KERNEL_FFI_HEADER_CXX}"
        INSTALL_COMMAND ""
        LOG_BUILD ON)

build_static_extension(${TARGET_NAME} ${EXTENSION_SOURCES})
build_loadable_extension(${TARGET_NAME} " " ${EXTENSION_SOURCES})

# TODO: when C linkage issue is resolved, we should switch back to using the generated headers
#include_directories(${DELTA_KERNEL_FFI_HEADER_PATH})

# Hides annoying linker warnings
set(CMAKE_OSX_DEPLOYMENT_TARGET 13.3 CACHE STRING "Minimum OS X deployment version" FORCE)

# Add the default client
add_compile_definitions(DEFINE_DEFAULT_ENGINE)

target_link_libraries(${LOADABLE_EXTENSION_NAME} "${CMAKE_BINARY_DIR}/libdelta_kernel_ffi.a")

install(
  TARGETS ${EXTENSION_NAME}
  EXPORT "${DUCKDB_EXPORT_SET}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}")
