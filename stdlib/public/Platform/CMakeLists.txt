set(swift_platform_name)
set(swift_platform_flags)
set(swift_platform_sources
    Platform.swift
    Misc.c
    TiocConstants.swift
    tgmath.swift.gyb)

add_swift_library(swiftDarwin ${SWIFT_STDLIB_LIBRARY_BUILD_TYPES} IS_SDK_OVERLAY
    Darwin.swift.gyb
    ${swift_platform_sources}
    POSIXError.swift
    MachError.swift

    SWIFT_COMPILE_FLAGS -Xfrontend -disable-objc-attr-requires-foundation-module "${SWIFT_RUNTIME_SWIFT_COMPILE_FLAGS}"
    LINK_FLAGS "${SWIFT_RUNTIME_SWIFT_LINK_FLAGS}"
    TARGET_SDKS ALL_APPLE_PLATFORMS
    API_NOTES_NON_OVERLAY)

add_swift_library(swiftGlibc ${SWIFT_STDLIB_LIBRARY_BUILD_TYPES} IS_SDK_OVERLAY
    Glibc.swift.gyb
    ${swift_platform_sources}

    SWIFT_COMPILE_FLAGS "${SWIFT_RUNTIME_SWIFT_COMPILE_FLAGS}"
    LINK_FLAGS "${SWIFT_RUNTIME_SWIFT_LINK_FLAGS}"
    TARGET_SDKS ANDROID FREEBSD LINUX
    DEPENDS glibc_modulemap)

add_swift_library(swiftMSVCRT ${SWIFT_STDLIB_LIBRARY_BUILD_TYPES} IS_SDK_OVERLAY
    msvcrt.swift
    ${swift_platform_sources}

    SWIFT_COMPILE_FLAGS "${SWIFT_RUNTIME_SWIFT_COMPILE_FLAGS}"
    LINK_FLAGS "${SWIFT_RUNTIME_SWIFT_LINK_FLAGS}"
    TARGET_SDKS WINDOWS)

add_swift_library(swiftNewlib ${SWIFT_STDLIB_LIBRARY_BUILD_TYPES} IS_SDK_OVERLAY
    Newlib.swift
    ${swift_platform_sources}

    SWIFT_COMPILE_FLAGS "${SWIFT_RUNTIME_SWIFT_COMPILE_FLAGS}"
    LINK_FLAGS "${SWIFT_RUNTIME_SWIFT_LINK_FLAGS}"
    TARGET_SDKS CYGWIN
    DEPENDS newlib_modulemap)

set(MINGW_SYSROOT "c:/msys64/mingw64")
add_swift_library(swiftMinGWCrt ${SWIFT_STDLIB_LIBRARY_BUILD_TYPES} IS_SDK_OVERLAY
    MinGWCrt.swift
    ${swift_platform_sources}

    SWIFT_COMPILE_FLAGS -Xcc --sysroot="${MINGW_SYSROOT}" "${SWIFT_RUNTIME_SWIFT_COMPILE_FLAGS}"
    LINK_FLAGS "${SWIFT_RUNTIME_SWIFT_LINK_FLAGS}"
    TARGET_SDKS MINGW
    DEPENDS mingw_crt_modulemap)

set(glibc_modulemap_target_list)
foreach(sdk ${SWIFT_SDKS})
  if(NOT "${sdk}" STREQUAL "LINUX" AND
     NOT "${sdk}" STREQUAL "FREEBSD" AND
     NOT "${sdk}" STREQUAL "ANDROID")
    continue()
  endif()

  foreach(arch ${SWIFT_SDK_${sdk}_ARCHITECTURES})
    set(arch_subdir "${SWIFT_SDK_${sdk}_LIB_SUBDIR}/${arch}")
    set(module_dir "${SWIFTLIB_DIR}/${arch_subdir}")

    # Determine the location of glibc headers based on the target.
    set(GLIBC_SYSROOT_RELATIVE_INCLUDE_PATH "/usr/include")
    set(GLIBC_SYSROOT_RELATIVE_ARCH_INCLUDE_PATH ${GLIBC_SYSROOT_RELATIVE_INCLUDE_PATH})

    # Some SDKs place their headers in architecture-specific subfolders.
    if((${sdk} STREQUAL "LINUX" OR ${sdk} STREQUAL "FREEBSD") AND CMAKE_LIBRARY_ARCHITECTURE)
      set(GLIBC_SYSROOT_RELATIVE_ARCH_INCLUDE_PATH "${GLIBC_SYSROOT_RELATIVE_ARCH_INCLUDE_PATH}/${CMAKE_LIBRARY_ARCHITECTURE}")
    endif()

    set(GLIBC_INCLUDE_PATH "${SWIFT_SDK_${sdk}_PATH}/${GLIBC_SYSROOT_RELATIVE_INCLUDE_PATH}")
    set(GLIBC_ARCH_INCLUDE_PATH "${SWIFT_SDK_${sdk}_PATH}/${GLIBC_SYSROOT_RELATIVE_ARCH_INCLUDE_PATH}")

    set(glibc_modulemap_source "glibc.modulemap.gyb")
    set(glibc_modulemap_out "${module_dir}/glibc.modulemap")

    # Configure the module map based on the target. Each platform needs to
    # reference different headers, based on what's available in their glibc.
    # This is the 'glibc.modulemap' in the 'resource-dir', so
    # it's the one we'll look at during the build process.
    handle_gyb_source_single(glibc_modulemap_target
        SOURCE "${glibc_modulemap_source}"
        OUTPUT "${glibc_modulemap_out}"
        FLAGS
            "-DCMAKE_SDK=${sdk}"
            "-DGLIBC_INCLUDE_PATH=${GLIBC_INCLUDE_PATH}"
            "-DGLIBC_ARCH_INCLUDE_PATH=${GLIBC_ARCH_INCLUDE_PATH}")

    list(APPEND glibc_modulemap_target_list ${glibc_modulemap_target})

    # If this SDK is a target for a non-native host, create a native modulemap
    # without a sysroot prefix. This is the one we'll install instead.
    if(NOT "${SWIFT_SDK_${SWIFT_HOST_VARIANT_SDK}_PATH}" STREQUAL "/")

      set(glibc_sysroot_relative_modulemap_out "${module_dir}/sysroot-relative-modulemaps/glibc.modulemap")
      handle_gyb_source_single(glibc_modulemap_native_target
          SOURCE "${glibc_modulemap_source}"
          OUTPUT "${glibc_sysroot_relative_modulemap_out}"
          FLAGS
              "-DCMAKE_SDK=${sdk}"
              "-DGLIBC_INCLUDE_PATH=${GLIBC_SYSROOT_RELATIVE_INCLUDE_PATH}"
              "-DGLIBC_ARCH_INCLUDE_PATH=${GLIBC_SYSROOT_RELATIVE_ARCH_INCLUDE_PATH}")

      list(APPEND glibc_modulemap_target_list ${glibc_modulemap_native_target})
      set(glibc_modulemap_out ${glibc_sysroot_relative_modulemap_out})
    endif()

    # FIXME: When SDK is a cross-compile target (SDK != Host), the generated
    #        modulemap will be relative to the Host, with hardcoded paths.
    #        It is not relocatable to the target platform itself.
    #        This only affects ANDROID right now, but could affect cross-compiled LINUX targets

    swift_install_in_component(sdk-overlay
        FILES "${glibc_modulemap_out}"
        DESTINATION "lib/swift/${arch_subdir}")

  endforeach()
endforeach()
add_custom_target(glibc_modulemap DEPENDS ${glibc_modulemap_target_list})

set(newlib_modulemap_target_list)
foreach(sdk ${SWIFT_SDKS})
  if(NOT "${sdk}" STREQUAL "CYGWIN")
    continue()
  endif()
  foreach(arch ${SWIFT_SDK_${sdk}_ARCHITECTURES})
    set(arch_subdir "${SWIFT_SDK_${sdk}_LIB_SUBDIR}/${arch}")
    set(module_dir "${SWIFTLIB_DIR}/${arch_subdir}")
    
    set(NEWLIB_INCLUDE_PATH "/usr/include")
    set(NEWLIB_ARCH_INCLUDE_PATH "/usr/include")
    
    set(newlib_modulemap_source "newlib.modulemap.gyb")
    set(newlib_modulemap_out "${module_dir}/newlib.modulemap")

    # Configure the module map based on the target.
    # This is the 'newlib.modulemap' in the 'resource-dir', so
    # it's the one we'll look at during the build process.
    handle_gyb_source_single(newlib_modulemap_target
        SOURCE "${newlib_modulemap_source}"
        OUTPUT "${newlib_modulemap_out}"
        FLAGS
            "-DCMAKE_SDK=${sdk}"
            "-DNEWLIB_INCLUDE_PATH=${NEWLIB_INCLUDE_PATH}"
            "-DNEWLIB_ARCH_INCLUDE_PATH=${NEWLIB_ARCH_INCLUDE_PATH}")

    list(APPEND newlib_modulemap_target_list ${newlib_modulemap_target})
            
    swift_install_in_component(sdk-overlay
        FILES "${newlib_modulemap_out}"
        DESTINATION "lib/swift/${arch_subdir}")
  endforeach()
endforeach()
add_custom_target(newlib_modulemap DEPENDS ${newlib_modulemap_target_list})

set(mingw_crt_modulemap_target_list)
foreach(sdk ${SWIFT_SDKS})
  if(NOT "${sdk}" STREQUAL "MINGW")
    continue()
  endif()
  foreach(arch ${SWIFT_SDK_${sdk}_ARCHITECTURES})
    set(arch_subdir "${SWIFT_SDK_${sdk}_LIB_SUBDIR}/${arch}")
    set(module_dir "${SWIFTLIB_DIR}/${arch_subdir}")
    
    set(MINGW_INCLUDE_PATH "${MINGW_SYSROOT}/include")
    set(DMINGW_ARCH_INCLUDE_PATH "${MINGW_SYSROOT}/x86_64-w64-mingw32/include")
    
    set(mingw_crt_modulemap_source "mingw_crt.modulemap.gyb")
    set(mingw_crt_modulemap_out "${module_dir}/mingw_crt.modulemap")

    # Configure the module map based on the target.
    # This is the 'mingw_crt.modulemap' in the 'resource-dir', so
    # it's the one we'll look at during the build process.
    handle_gyb_source_single(mingw_crt_modulemap_target
        SOURCE "${mingw_crt_modulemap_source}"
        OUTPUT "${mingw_crt_modulemap_out}"
        FLAGS
            "-DMINGW_INCLUDE_PATH=${MINGW_INCLUDE_PATH}"
            "-DMINGW_ARCH_INCLUDE_PATH=${DMINGW_ARCH_INCLUDE_PATH}")

    list(APPEND mingw_crt_modulemap_target_list ${mingw_crt_modulemap_target})
            
    swift_install_in_component(sdk-overlay
        FILES "${mingw_crt_modulemap_out}"
        DESTINATION "lib/swift/${arch_subdir}")
  endforeach()
endforeach()
add_custom_target(mingw_crt_modulemap DEPENDS ${mingw_crt_modulemap_target_list})
