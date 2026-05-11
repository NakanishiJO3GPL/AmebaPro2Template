cmake_minimum_required(VERSION 3.16)

project(app_tz)

# =============================================================================
# Paths
# =============================================================================
set(prj_root "${CMAKE_CURRENT_SOURCE_DIR}/..")
set(sdk_root "${CMAKE_CURRENT_SOURCE_DIR}/../vendor")

message(STATUS "Build libraries : ${BUILD_LIB}")
message(STATUS "Build FPGA      : ${BUILD_FPGA}")
message(STATUS "Build PXP       : ${BUILD_PXP}")

# =============================================================================
# Build configuration
# =============================================================================
include(${prj_root}/CMakeFiles/config.cmake)
include(${prj_root}/CMakeFiles/sensor_iq.cmake)

# =============================================================================
# Driver sources and application target
# (driver_s.cmake builds the TZ secure side; driver.cmake builds the main app)
# =============================================================================
if(BUILD_TZ)
  include(./driver_s.cmake)
endif()
include(./driver.cmake)

# =============================================================================
# Tool: ELF2BIN
# =============================================================================
if(NOT DEFINED ELF2BIN)
  if(LINUX)
    set(ELF2BIN ${prj_root}/tools/elf2bin.linux)
  else()
    set(ELF2BIN ${prj_root}/tools/elf2bin.exe)
  endif()
endif()

# =============================================================================
# Firmware JSON configuration
# =============================================================================
if(MPCHIP)
  if(NOT DEFINED POSTBUILD_FW_NTZ)
    set(POSTBUILD_FW_NTZ ${prj_root}/tools/amebapro2_firmware_ntz.json)
  endif()
  if(NOT DEFINED POSTBUILD_FW_TZ)
    set(POSTBUILD_FW_TZ  ${prj_root}/tools/amebapro2_firmware_tz.json)
  endif()

  if(BUILD_TZ)
    set(postbuild_cfg ${POSTBUILD_FW_TZ})
    set(firmware_name firmware_tz.bin)
  else()
    set(postbuild_cfg ${POSTBUILD_FW_NTZ})
    set(firmware_name firmware_ntz.bin)
  endif()
else()
  if(BUILD_TZ)
    set(postbuild_cfg ${prj_root}/tools/amebapro2_firmware.json)
    set(firmware_name firmware_tz.bin)
  else()
    set(postbuild_cfg ${prj_root}/tools/amebapro2_firmware_sns.json)
    set(firmware_name firmware_ntz.bin)
  endif()
endif()

# =============================================================================
# Link dependencies
# =============================================================================
set_target_properties(${app} PROPERTIES
  LINK_DEPENDS "${ELF2BIN};${postbuild_cfg}"
)

# =============================================================================
# Post-build: generate firmware binary and copy to output
# =============================================================================
add_dependencies(${app} fcs_isp_iq)

add_custom_command(TARGET ${app} POST_BUILD
  # Copy VOE binary
  COMMAND ${CMAKE_COMMAND} -E copy
    ${sdk_root}/soc/8735b/fwlib/rtl8735b/lib/source/ram/video/voe_bin/voe.bin
    voe.bin

  # Convert ELF to firmware binary
  COMMAND ${CMAKE_COMMAND} -E copy ${postbuild_cfg} amebapro2_firmware.json
  COMMAND ${ELF2BIN} convert amebapro2_firmware.json FIRMWARE ${firmware_name}

  # Copy artifacts to output directory
  COMMAND ${CMAKE_COMMAND} -E make_directory output
  COMMAND ${CMAKE_COMMAND} -E copy ${firmware_name} output
)
