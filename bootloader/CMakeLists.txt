cmake_minimum_required(VERSION 3.16)

project(bootloader)

# =============================================================================
# Paths
# =============================================================================
set(prj_root "${CMAKE_CURRENT_SOURCE_DIR}/..")
set(sdk_root "${CMAKE_CURRENT_SOURCE_DIR}/../vendor")

# =============================================================================
# Common configuration
# =============================================================================
include(${prj_root}/CMakeFiles/config.cmake)

# =============================================================================
# Compile definition flags
# =============================================================================
set(boot_flags
  CONFIG_BUILD_BOOT=1
  CONFIG_BUILD_RAM=1
  CONFIG_BUILD_LIB=1
  CONFIG_PLATFORM_8735B
  CONFIG_RTL8735B_PLATFORM=1
)

# =============================================================================
# Include paths
# =============================================================================
include(${prj_root}/CMakeFiles/includepath.cmake)

set(boot_include_dirs
  ${inc_path}
  ${prj_root}/rtos/freertos/${freertos}/Source/portable/GCC/ARM_CM33_NTZ/non_secure
  ${sdk_root}/soc/8735b/fwlib/rtl8735b/lib/source/ram/video
  ${sdk_root}/soc/8735b/fwlib/rtl8735b/lib/source/ram/video/enc/inc
)

# =============================================================================
# BootFCS library
# =============================================================================
set(fcs_sources
  ${sdk_root}/video/driver/RTL8735B/video_user_boot.c
)

add_library(bootfcs ${fcs_sources})
target_include_directories(bootfcs PUBLIC ${boot_include_dirs})
target_compile_definitions(bootfcs PRIVATE ${boot_flags})
target_compile_options(bootfcs PRIVATE ${WARN_ERR_FLAGS})

# =============================================================================
# Bootloader executable
# =============================================================================
set(bootloader_target bootloader)

set(bootloader_sources
  ${sdk_root}/video/driver/RTL8735B/video_boot.c
  ${sdk_root}/soc/8735b/misc/platform/user_boot.c
)

add_executable(${bootloader_target}
  ${bootloader_sources}
  $<TARGET_OBJECTS:bootfcs>
)
target_include_directories(${bootloader_target} PUBLIC ${boot_include_dirs})
target_compile_definitions(${bootloader_target} PRIVATE ${boot_flags})
target_compile_options(${bootloader_target} PRIVATE ${WARN_ERR_FLAGS})

# =============================================================================
# Imported libraries
# =============================================================================
add_library(hal_pmc_lib STATIC IMPORTED)
set_target_properties(hal_pmc_lib PROPERTIES
  IMPORTED_LOCATION ${sdk_root}/soc/8735b/fwlib/rtl8735b/lib/lib/hal_pmc.a
)

# =============================================================================
# Link settings
# =============================================================================
target_link_directories(${bootloader_target} PRIVATE ${prj_root}/bootloader/output)

target_link_libraries(${bootloader_target}
  -Wl,--whole-archive
  boot
  hal_pmc_lib
)

set(ld_script "${CMAKE_CURRENT_SOURCE_DIR}/rtl8735b_boot_mp.ld")

target_link_options(${bootloader_target} PUBLIC
  "LINKER:SHELL:-L ${sdk_root}/soc/8735b/cmsis/rtl8735b/source/GCC"
  "LINKER:SHELL:-T ${ld_script}"
  "LINKER:SHELL:-Map=${CMAKE_CURRENT_BINARY_DIR}/bootloader.map"
  "LINKER:-wrap,puts"
  "LINKER:-wrap,printf"
  "LINKER:-wrap,sprintf"
  "LINKER:-wrap,snprintf"
  "LINKER:-wrap,vsnprintf"
  "LINKER:-wrap,vprintf"
)

set_target_properties(${bootloader_target} PROPERTIES
  LINK_DEPENDS "${ld_script};${ELF2BIN};${POSTBUILD_BOOT}"
)

# =============================================================================
# Post-build commands
# =============================================================================
add_custom_command(TARGET ${bootloader_target} POST_BUILD
  # Generate symbol map, disassembly, and copy ELF
  COMMAND ${CMAKE_NM} $<TARGET_FILE:${bootloader_target}> | sort > ${bootloader_target}.nm.map
  COMMAND ${CMAKE_OBJEDUMP} -d $<TARGET_FILE:${bootloader_target}> > ${bootloader_target}.asm
  COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${bootloader_target}> ${bootloader_target}.axf

  # Generate boot.bin
  COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_BOOT} amebapro2_bootloader.json
  COMMAND ${ELF2BIN} convert amebapro2_bootloader.json BOOTLOADER boot.bin

  # Generate boot_fcs.bin
  COMMAND echo $<TARGET_OBJECTS:bootfcs>
  COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_OBJECTS:bootfcs> tmp.o
  COMMAND ${CMAKE_OBJECOPY} -O binary tmp.o boot_fcs.bin -j .data.video_boot_stream
  COMMAND ${CHKSUM} -m fcs ./boot_fcs.bin

  # Copy artifacts to output directory
  COMMAND ${CMAKE_COMMAND} -E remove_directory output
  COMMAND ${CMAKE_COMMAND} -E make_directory output
  COMMAND ${CMAKE_COMMAND} -E copy boot.bin output
  COMMAND ${CMAKE_COMMAND} -E copy boot_fcs.bin output
  COMMAND ${CMAKE_COMMAND} -E copy ${bootloader_target}.nm.map output
  COMMAND ${CMAKE_COMMAND} -E copy ${bootloader_target}.asm output
  COMMAND ${CMAKE_COMMAND} -E copy ${bootloader_target}.axf output
)
