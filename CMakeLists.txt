cmake_minimum_required(VERSION 3.16)

project(flash_bin)

# =============================================================================
# Paths
# =============================================================================
set(prj_root "${CMAKE_CURRENT_SOURCE_DIR}")
set(sdk_root "${CMAKE_CURRENT_SOURCE_DIR}/vendor")

include(${prj_root}/CMakeFiles/config.cmake)

# =============================================================================
# Copy NN model files to build directory
# =============================================================================
file(GLOB NETWORK_BINARY "${NN_MODEL_PATH}/*.nb")
file(COPY ${NETWORK_BINARY} DESTINATION .)

# =============================================================================
# Bootloader configuration
# =============================================================================
if(EXISTS "${prj_root}/bootloader/CMakeLists.txt")
  message(STATUS "build bootloader")
  add_subdirectory(bootloader)
  set(boot_image bootloader/boot.bin)
  set(fcs_param  bootloader/boot_fcs.bin)
  set(boot_dep   bootloader)
else()
  message(STATUS "use released boot.bin")
  set(boot_image ${prj_root}/bootloader/boot.bin)
  set(fcs_param  ${prj_root}/bootloader/boot_fcs.bin)
  set(boot_dep   ${ELF2BIN})
endif()

# =============================================================================
# Firmware configuration (TrustZone vs Non-TrustZone)
# =============================================================================
if(BUILD_TZ)
  if(BUILD_WLANMP)
    set(target flash_tz_mp)
  else()
    set(target flash_tz)
  endif()
  set(fw      drivers/firmware_tz.bin)
  set(app_dep application.s application.ns)
  set(symbols drivers/application.ns.symbols)
else()
  if(BUILD_WLANMP)
    set(target flash_ntz_mp)
  else()
    set(target flash_ntz)
  endif()
  set(fw      drivers/firmware_ntz.bin)
  set(app_dep application.ntz)
  set(symbols drivers/application.ntz.symbols)
endif()

# Common regardless of TZ setting
set(isp_iq  drivers/firmware_isp_iq.bin)
set(bttrace drivers/APP.trace)

# Application target name (must match the name set in driver.cmake)
if(BUILD_TZ)
  set(app application.ns)
else()
  set(app application.ntz)
endif()

# =============================================================================
# Scenario / application layer
# =============================================================================
# scenario.cmake appends to app_sources (MMF / NN / AINR modules) and populates
# scn_sources (models, main.c, utils), scn_flags, scn_inc_path, scn_libs.
#
# Including it BEFORE add_subdirectory(drivers) is intentional:
# CMake copies the parent scope into the child at add_subdirectory time, so
# app_sources already contains the scenario additions when driver.cmake calls
# add_executable().  scn_sources / scn_flags / scn_inc_path / scn_libs are
# applied to the target afterwards via target_*() (targets are global).
if(DEFINED SCENARIO AND SCENARIO AND NOT "${SCENARIO}" STREQUAL "standard")
  message(STATUS "SCENARIO = ${SCENARIO}")
  if(EXISTS ${prj_root}/scenario/${SCENARIO}/scenario.cmake)
    message(STATUS "Building scenario: ${SCENARIO}")
    include(${prj_root}/scenario/${SCENARIO}/scenario.cmake)
  else()
    message(FATAL_ERROR "SCENARIO '${SCENARIO}' not found")
  endif()
  if(NOT DEBUG)
    set(SCENARIO OFF CACHE STRING INTERNAL FORCE)
  endif()
else()
  message(STATUS "Building standard scenario")
  include(${prj_root}/app/scenario.cmake)
endif()

add_subdirectory(drivers)

# Attach scenario-specific content to the app target created in drivers/.
target_sources(${app}          PRIVATE ${scn_sources})
target_compile_definitions(${app} PRIVATE ${scn_flags})
target_include_directories(${app} PUBLIC  ${scn_inc_path})
target_link_libraries(${app}            ${scn_libs})

# =============================================================================
# Partition lists for ELF2BIN combine (must be a single argument, no spaces)
# =============================================================================
set(combine_flash
  "PT_PT=partition.bin,CER_TBL=certable.bin,KEY_CER1=certificate.bin,PT_BL_PRI=boot.bin,PT_FW1=firmware.bin,PT_ISP_IQ=firmware_isp_iq.bin,PT_FCSDATA=boot_fcs.bin"
)
set(combine_flash_nn
  "PT_PT=partition.bin,CER_TBL=certable.bin,KEY_CER1=certificate.bin,PT_BL_PRI=boot.bin,PT_FW1=firmware.bin,PT_NN_MDL=nn_model.bin,PT_ISP_IQ=firmware_isp_iq.bin,PT_FCSDATA=boot_fcs.bin"
)
set(combine_hash
  "PT_PT=partition_hashed.bin,CER_TBL=certable.bin,KEY_CER1=certificate_signed.bin,PT_BL_PRI=boot_hashed.bin,PT_FW1=firmware_hashed.bin,PT_ISP_IQ=firmware_isp_iq.bin,PT_FCSDATA=boot_fcs.bin"
)
set(combine_sign
  "PT_PT=partition_signed.bin,CER_TBL=certable.bin,KEY_CER1=certificate_signed.bin,PT_BL_PRI=boot_signed.bin,PT_FW1=firmware_signed.bin,PT_ISP_IQ=firmware_isp_iq.bin,PT_FCSDATA=boot_fcs.bin"
)
set(combine_sign_enc
  "PT_PT=partition_signed.bin,CER_TBL=certable.bin,KEY_CER1=certificate_signed.bin,PT_BL_PRI=boot_signed_enc.bin,PT_FW1=firmware_signed_enc.bin,PT_ISP_IQ=firmware_isp_iq.bin,PT_FCSDATA=boot_fcs.bin"
)

# ---------------------------------------------------------------------------
# Key generation
# ---------------------------------------------------------------------------

# File-level output (used as dependency by plain_img)
add_custom_command(
  OUTPUT key_public.json key_private.json
  COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_KEY_CFG} keycfg.json
  COMMAND ${ELF2BIN} keygen keycfg.json key
)

# Manually invocable target
add_custom_target(keygen
  COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_KEY_CFG} keycfg.json
  COMMAND ${ELF2BIN} keygen keycfg.json key
  DEPENDS ${ELF2BIN}
)

# ---------------------------------------------------------------------------
# plain_img: assemble all binary components
# ---------------------------------------------------------------------------
add_custom_target(plain_img
  # Certificate and partition table
  COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_CERT} certificate.json
  COMMAND ${ELF2BIN} convert certificate.json CERT_TABLE  certable.bin
  COMMAND ${ELF2BIN} convert certificate.json CERTIFICATE certificate.bin
  COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_PART} amebapro2_partitiontable.json
  COMMAND ${CMAKE_COMMAND} -E touch user.bin  # placeholder required by amebapro2_partitiontable.json
  COMMAND ${ELF2BIN} convert amebapro2_partitiontable.json PARTITIONTABLE partition.bin

  # Binary files
  COMMAND ${CMAKE_COMMAND} -E copy ${boot_image}  .
  COMMAND ${CMAKE_COMMAND} -E copy ${fcs_param}   .
  COMMAND ${CMAKE_COMMAND} -E copy ${fw}          ./firmware.bin
  COMMAND ${CMAKE_COMMAND} -E copy ${isp_iq}      ./firmware_isp_iq.bin
  COMMAND ${CMAKE_COMMAND} -E copy ${symbols}     ./application.symbols
  COMMAND ${CMAKE_COMMAND} -E copy ${bttrace}     ./APP.trace

  DEPENDS ${boot_dep}
  DEPENDS ${app_dep}
  DEPENDS ${POSTBUILD_CERT}
  DEPENDS ${POSTBUILD_KEY_CFG}
  DEPENDS ${POSTBUILD_PART}
  DEPENDS key_public.json
  DEPENDS key_private.json
  DEPENDS ${ELF2BIN}
)

# ---------------------------------------------------------------------------
# flash: combine binaries and generate OTA images
# ---------------------------------------------------------------------------
add_custom_target(flash
  COMMAND ${ELF2BIN} combine amebapro2_partitiontable.json ${target}.bin ${combine_flash}

  # OTA images
  COMMAND ${CMAKE_COMMAND} -E copy ./firmware.bin        ./ota.bin
  COMMAND ${CHKSUM} ./ota.bin
  COMMAND ${CMAKE_COMMAND} -E copy ./firmware_isp_iq.bin ./isp_iq_ota.bin
  COMMAND ${CHKSUM} ./isp_iq_ota.bin
  COMMAND ${CMAKE_COMMAND} -E copy ./boot.bin            ./boot_ota.bin
  COMMAND ${CHKSUM} ./boot_ota.bin

  DEPENDS plain_img
  DEPENDS ${ELF2BIN}
)

# ---------------------------------------------------------------------------
# auto_model_cfg: configure NN model
# ---------------------------------------------------------------------------
if(UNITEST)
  add_custom_target(auto_model_cfg
    COMMAND ${CMAKE_COMMAND} -E cmake_echo_color --cyan "auto NN model config start"
    COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_FWFS_NN} amebapro2_fwfs_nn_models.json
    COMMAND ${NNMDLCFG} amebapro2_fwfs_nn_models.json application.symbols
    DEPENDS plain_img
    DEPENDS ${POSTBUILD_FWFS_NN}
  )
else()
  add_custom_target(auto_model_cfg
    COMMAND ${CMAKE_COMMAND} -E cmake_echo_color --cyan "skip model config"
    COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_FWFS_NN} amebapro2_fwfs_nn_models.json
    DEPENDS plain_img
    DEPENDS ${POSTBUILD_FWFS_NN}
  )
endif()

# ---------------------------------------------------------------------------
# flash_nn: combine binaries with NN model and generate OTA images
# ---------------------------------------------------------------------------
add_custom_target(flash_nn
  COMMAND ${CMAKE_COMMAND} -E copy ${POSTBUILD_NNMDL} amebapro2_nn_model.json
  COMMAND ${ELF2BIN} convert amebapro2_fwfs_nn_models.json FWFS    fwfs_nn_model.bin
  COMMAND ${ELF2BIN} convert amebapro2_nn_model.json       FIRMWARE nn_model.bin
  COMMAND ${ELF2BIN} combine amebapro2_partitiontable.json ${target}.nn.bin ${combine_flash_nn}

  # OTA images
  COMMAND ${CMAKE_COMMAND} -E copy ./firmware.bin        ./ota.bin
  COMMAND ${CHKSUM} ./ota.bin
  COMMAND ${CMAKE_COMMAND} -E copy ./nn_model.bin        ./nn_model_ota.bin
  COMMAND ${CHKSUM} ./nn_model_ota.bin
  COMMAND ${CMAKE_COMMAND} -E copy ./firmware_isp_iq.bin ./isp_iq_ota.bin
  COMMAND ${CHKSUM} ./isp_iq_ota.bin

  DEPENDS plain_img
  DEPENDS auto_model_cfg
  DEPENDS ${ELF2BIN}
  DEPENDS ${POSTBUILD_NNMDL}
  DEPENDS ${POSTBUILD_FWFS_NN}
)

# ---------------------------------------------------------------------------
# Secure image targets (hash / sign / sign+encrypt)
# ---------------------------------------------------------------------------
add_custom_target(hash
  COMMAND ${ELF2BIN} secure sign+dbg=cert key_private.json key_public.json certificate.bin certificate_signed.bin
  COMMAND ${ELF2BIN} secure hash+dbg=ptab key_private.json key_public.json partition.bin   partition_hashed.bin
  COMMAND ${ELF2BIN} secure hash+dbg=boot key_private.json key_public.json boot.bin        boot_hashed.bin
  COMMAND ${ELF2BIN} secure hash+dbg=fw   key_private.json key_public.json firmware.bin    firmware_hashed.bin
  COMMAND ${ELF2BIN} combine amebapro2_partitiontable.json ${target}.bin ${combine_hash}
  COMMAND ${CMAKE_COMMAND} -E copy ./firmware_hashed.bin ./ota.bin
  COMMAND ${CHKSUM} ./ota.bin
  DEPENDS plain_img
  DEPENDS ${ELF2BIN}
)

add_custom_target(sign
  COMMAND ${ELF2BIN} secure sign+dbg=cert      key_private.json key_public.json certificate.bin certificate_signed.bin
  COMMAND ${ELF2BIN} secure sign+hash+dbg=ptab key_private.json key_public.json partition.bin   partition_signed.bin
  COMMAND ${ELF2BIN} secure sign+hash+dbg=boot key_private.json key_public.json boot.bin        boot_signed.bin
  COMMAND ${ELF2BIN} secure sign+hash+dbg=fw   key_private.json key_public.json firmware.bin    firmware_signed.bin
  COMMAND ${ELF2BIN} combine amebapro2_partitiontable.json ${target}.bin ${combine_sign}
  COMMAND ${CMAKE_COMMAND} -E copy ./firmware_signed.bin ./ota.bin
  COMMAND ${CHKSUM} ./ota.bin
  DEPENDS plain_img
  DEPENDS ${ELF2BIN}
)

add_custom_target(sign_enc
  COMMAND ${ELF2BIN} secure sign+dbg=cert           key_private.json key_public.json certificate.bin       certificate_signed.bin
  COMMAND ${ELF2BIN} secure sign+hash+dbg=ptab      key_private.json key_public.json partition.bin         partition_signed.bin
  COMMAND ${ELF2BIN} secure sign+enc+hash+dbg=boot  key_private.json key_public.json ${POSTBUILD_ENC_BOOT} boot.bin     boot_signed_enc.bin
  COMMAND ${ELF2BIN} secure sign+enc+hash+dbg=fw    key_private.json key_public.json ${POSTBUILD_ENC_FW}   firmware.bin firmware_signed_enc.bin
  COMMAND ${ELF2BIN} combine amebapro2_partitiontable.json ${target}.bin ${combine_sign_enc}
  COMMAND ${CMAKE_COMMAND} -E copy ./firmware_signed_enc.bin ./ota.bin
  COMMAND ${CHKSUM} ./ota.bin
  DEPENDS plain_img
  DEPENDS ${ELF2BIN}
)
