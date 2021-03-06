# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License.

# NOTE: Because we still build mbed TLS via `ExternalProject_Add`, it
# does not automatically inherit our build settings (nor could it
# because it doesn't link against our targets). Unfortunately, that
# means we have to maintain these flags to be consistent with
# those used to build `oecore`.
list(APPEND MBEDTLS_WRAP_CFLAGS_LIST
  ${OE_SPECTRE_MITIGATION_FLAGS}
  -nostdinc
  -I${PROJECT_SOURCE_DIR}/include/openenclave/corelibc
  -I${PROJECT_SOURCE_DIR}/include
  -DOE_NEED_STDC_NAMES
  -fPIE
  -ftls-model=local-exec
  -fvisibility=hidden
  -fno-stack-protector
  -fno-builtin-udivti3)

if (USE_CLANGW)
  list(APPEND MBEDTLS_WRAP_CFLAGS_LIST "-target x86_64-pc-linux")
  # Setup clang wrapper toolchain
  set(MBEDTLS_TOOLCHAIN "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_CURRENT_SOURCE_DIR}/toolchain-clangw.cmake")

  # Setup linker
  find_program(LLVM_AR "llvm-ar.exe")
  set(MBEDTLS_LINKER "-DCMAKE_AR=${LLVM_AR}")
  set(MBEDTLS_COMPILER "")
  set(CMAKE_AR ${LLVM_AR})
else()
  set(MBEDTLS_TOOLCHAIN "")
  set(MBEDTLS_LINKER "")
  set(MBEDTLS_COMPILER "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}")
endif()

# Create a patched version of mbed TLS's `config.h` that the external
# project depends on.
string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE)
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")
  set(MBEDTLS_CERTS_C TRUE)
  set(MBEDTLS_DEBUG_C TRUE)
endif ()

list(JOIN MBEDTLS_WRAP_CFLAGS_LIST " " MBEDTLS_WRAP_CFLAGS)
configure_file(config.h config.h) # This copies from source to binary folders.

# Since we define mbedtls to use an alternate entropy source, it uses an
# undefined mebdtls_hardware_poll function. We define it here and add it to
# the mbedcrypto library, so that we avoid link errors and also circular
# library dependecies.
add_library(oe_mbedtls_hardware_poll OBJECT
  mbedtls_hardware_poll.c)
maybe_build_using_clangw(oe_mbedtls_hardware_poll)
target_link_libraries(oe_mbedtls_hardware_poll oecore)

include(ExternalProject)
ExternalProject_Add(mbedtls-wrap
  DEPENDS oecore oe_includes oe_mbedtls_hardware_poll

  DOWNLOAD_COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${CMAKE_CURRENT_LIST_DIR}/mbedtls <SOURCE_DIR>

  UPDATE_COMMAND ${CMAKE_COMMAND} -E copy_if_different
    ${CMAKE_CURRENT_BINARY_DIR}/config.h <SOURCE_DIR>/include/mbedtls/config.h

  CMAKE_ARGS
    ${MBEDTLS_TOOLCHAIN}
    ${MBEDTLS_LINKER}
    ${MBEDTLS_COMPILER}
    -DCMAKE_C_FLAGS=${MBEDTLS_WRAP_CFLAGS}
    -DENABLE_PROGRAMS=OFF
    -DENABLE_TESTING=OFF
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}

  # This is a no-op to override the default.
  INSTALL_COMMAND ""
  BYPRODUCTS
    <BINARY_DIR>/library/libmbedtls.a
    <BINARY_DIR>/library/libmbedx509.a
    <BINARY_DIR>/library/libmbedcrypto.a)

ExternalProject_Get_property(mbedtls-wrap SOURCE_DIR)
ExternalProject_Get_property(mbedtls-wrap BINARY_DIR)

# Add the mbedtls_hardware_poll function to the mbedcrypto library.
add_custom_command(TARGET mbedtls-wrap
    POST_BUILD
    COMMAND ${CMAKE_AR} r ${BINARY_DIR}/library/libmbedcrypto.a $<TARGET_OBJECTS:oe_mbedtls_hardware_poll>)

# NOTE: These imported libraries must be specified before `oecore`
# when linking, hence their dependency on it.
add_library(mbedtls_imported IMPORTED STATIC)
set_target_properties(mbedtls_imported PROPERTIES
  IMPORTED_LOCATION ${BINARY_DIR}/library/libmbedtls.a)
target_link_libraries(mbedtls_imported INTERFACE oecore)

add_library(mbedx509_imported IMPORTED STATIC)
set_target_properties(mbedx509_imported PROPERTIES
  IMPORTED_LOCATION ${BINARY_DIR}/library/libmbedx509.a)
target_link_libraries(mbedx509_imported INTERFACE oecore)

add_library(mbedcrypto_imported IMPORTED STATIC)
set_target_properties(mbedcrypto_imported PROPERTIES
  IMPORTED_LOCATION ${BINARY_DIR}/library/libmbedcrypto.a)
target_link_libraries(mbedcrypto_imported INTERFACE oecore)

# Convenience lib for use in target_link_libraries
add_library(mbedcrypto INTERFACE)

add_dependencies(mbedcrypto mbedtls-wrap)

target_include_directories(mbedcrypto INTERFACE
  $<BUILD_INTERFACE:${SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_INCLUDEDIR}/openenclave/3rdparty>)

target_link_libraries(mbedcrypto INTERFACE
  $<BUILD_INTERFACE:mbedtls_imported>
  $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_LIBDIR}/openenclave/enclave/libmbedtls.a>
  $<BUILD_INTERFACE:mbedx509_imported>
  $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_LIBDIR}/openenclave/enclave/libmbedx509.a>
  $<BUILD_INTERFACE:mbedcrypto_imported>
  $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_LIBDIR}/openenclave/enclave/libmbedcrypto.a>
  # NOTE: This dependency is repeated because during installation the
  # imported libraries are not part of the CMake dependency graph, so
  # this transitive dependency would be left out.
  oecore
  oe_includes)

install(TARGETS mbedcrypto EXPORT openenclave-targets)

install(DIRECTORY ${SOURCE_DIR}/include/mbedtls
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/openenclave/3rdparty)

# NOTE: Because these libraries are `IMPORTED`, it is not possible
# with CMake to install them as targets, so instead they must be
# installed as files. This is what causes the above problem with the
# repeated dependency on `oecore`.
install(FILES ${BINARY_DIR}/library/libmbedtls.a
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/openenclave/enclave)

install(FILES ${BINARY_DIR}/library/libmbedx509.a
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/openenclave/enclave)

install(FILES ${BINARY_DIR}/library/libmbedcrypto.a
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/openenclave/enclave)
