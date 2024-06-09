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

# Propagate arch to rust build for CI
set(RUST_PLATFORM_TARGET "wasm32-unknown-unknown")

# Add rust_example as a CMake target
ExternalProject_Add(
        ${KERNEL_NAME}
        GIT_REPOSITORY "https://github.com/carlopi/delta-kernel-rs"
        GIT_TAG main
        CONFIGURE_COMMAND ""
        UPDATE_COMMAND ""
        BUILD_IN_SOURCE 1
        # Build release build
        BUILD_COMMAND cargo clippy --package delta_kernel_ffi --workspace --release --target=${RUST_PLATFORM_TARGET}
        # Build DATs
        COMMAND cargo build --manifest-path=${CMAKE_BINARY_DIR}/rust/src/delta_kernel/acceptance/Cargo.toml --target=${RUST_PLATFORM_TARGET}
        BUILD_BYPRODUCTS "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/debug/libdelta_kernel_ffi.a"
        BUILD_BYPRODUCTS "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/release/libdelta_kernel_ffi.a"
        BUILD_BYPRODUCTS "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/ffi-headers/delta_kernel_ffi.h"
        BUILD_BYPRODUCTS "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/ffi-headers/delta_kernel_ffi.hpp"
        INSTALL_COMMAND ""
        LOG_BUILD ON)

build_static_extension(${TARGET_NAME} ${EXTENSION_SOURCES})
build_loadable_extension(${TARGET_NAME} " " ${EXTENSION_SOURCES})

include_directories(${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/ffi-headers)
include_directories(${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/ffi-headers)

# Hides annoying linker warnings
set(CMAKE_OSX_DEPLOYMENT_TARGET 13.3 CACHE STRING "Minimum OS X deployment version" FORCE)

# Add the default client
add_compile_definitions(DEFINE_DEFAULT_ENGINE)

# Link delta-kernal-rs to static lib
target_link_libraries(${EXTENSION_NAME}
        debug "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/debug/libdelta_kernel_ffi.a"
        optimized "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/release/libdelta_kernel_ffi.a"
        ${PLATFORM_LIBS})
add_dependencies(${EXTENSION_NAME} delta_kernel)

# Link delta-kernal-rs to dynamic lib
target_link_libraries(${LOADABLE_EXTENSION_NAME}
        debug "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/debug/libdelta_kernel_ffi.a"
        optimized "${CMAKE_BINARY_DIR}/rust/src/delta_kernel/target/${RUST_PLATFORM_TARGET}/release/libdelta_kernel_ffi.a"
        ${PLATFORM_LIBS})
add_dependencies(${LOADABLE_EXTENSION_NAME} delta_kernel)

install(
  TARGETS ${EXTENSION_NAME}
  EXPORT "${DUCKDB_EXPORT_SET}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}")
