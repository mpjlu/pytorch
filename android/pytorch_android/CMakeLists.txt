cmake_minimum_required(VERSION 3.4.1)
project(pytorch CXX)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_VERBOSE_MAKEFILE ON)

set(pytorch_android_DIR ${CMAKE_CURRENT_LIST_DIR}/src/main/cpp)

if (ANDROID_ABI)
  set(libtorch_include_DIR ${pytorch_android_DIR}/libtorch_include/${ANDROID_ABI})
  set(BUILD_SUBDIR ${ANDROID_ABI})
else()
  if (NOT LIBTORCH_HOME)
    message(FATAL_ERROR
      "pytorch_android requires LIBTORCH_HOME to be defined for non-Android builds.")
  endif()
  set(libtorch_include_DIR ${LIBTORCH_HOME}/include)
  link_directories(${LIBTORCH_HOME}/lib)
  set(BUILD_SUBDIR host)
endif()

message(STATUS "libtorch dir:${libtorch_DIR}")

file(GLOB pytorch_android_SOURCES
  ${pytorch_android_DIR}/*.cpp
)

add_library(pytorch SHARED
    ${pytorch_android_SOURCES}
)

target_compile_options(pytorch PRIVATE
  -fexceptions
)

target_include_directories(pytorch PUBLIC
    ${libtorch_include_DIR}
)

set(fbjni_DIR ${CMAKE_CURRENT_LIST_DIR}/../libs/fbjni/)
set(fbjni_BUILD_DIR ${CMAKE_BINARY_DIR}/fbjni/${BUILD_SUBDIR})

add_subdirectory(${fbjni_DIR} ${fbjni_BUILD_DIR})

if (ANDROID_ABI)

  function(import_static_lib name)
    add_library(${name} STATIC IMPORTED)
    set_property(
        TARGET ${name}
        PROPERTY IMPORTED_LOCATION
        ${CMAKE_CURRENT_LIST_DIR}/src/main/jniLibs/${ANDROID_ABI}/${name}.a)
  endfunction(import_static_lib)

  import_static_lib(libtorch)
  import_static_lib(libc10)
  import_static_lib(libnnpack)
  import_static_lib(libpytorch_qnnpack)
  import_static_lib(libeigen_blas)
  import_static_lib(libcpuinfo)
  import_static_lib(libclog)

    # Link most things statically on Android.
  target_link_libraries(pytorch
      fbjni
      -Wl,--gc-sections
      -Wl,--whole-archive
      libtorch
      -Wl,--no-whole-archive
      libc10
      libnnpack
      libpytorch_qnnpack
      libeigen_blas
      libcpuinfo
      libclog
  )

else()

  # Prefer dynamic linking on the host
  target_link_libraries(pytorch
      fbjni
      -Wl,--gc-sections
      -Wl,--whole-archive
      torch
      -Wl,--no-whole-archive
      c10
      nnpack
      pytorch_qnnpack
      cpuinfo
      clog
  )

endif()
