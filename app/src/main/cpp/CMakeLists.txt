cmake_minimum_required(VERSION 3.6)

project (iamf)
message(STATUS "start build iamf")
#message(WARNING "${PROJECT_SOURCE_DIR}")

#set(PROJECT_SOURCE_DIR "${CMAKE_SOURCE_DIR}")
SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3 -Werror=unused-variable")
SET(CMAKE_POSITION_INDEPENDENT_CODE ON)

option(BUILD_SHARED_LIBS "Build shared library" ON)

set(CODEC_LIB_DIR  "${PROJECT_SOURCE_DIR}/dep_codecs/lib/${ANDROID_ABI}")
set(CODEC_INCLUDE_DIR  "${PROJECT_SOURCE_DIR}/dep_codecs/include")
set(EXTER_LIB_DIR  "${PROJECT_SOURCE_DIR}/dep_external/lib")
set(EXTER_INCLUDE_DIR  "${PROJECT_SOURCE_DIR}/dep_external/include")

set(CMAKE_INSTALL_PREFIX "${PROJECT_SOURCE_DIR}/build_libs")
file(MAKE_DIRECTORY ${CMAKE_INSTALL_PREFIX})

option(SUPPORT_VERIFIER "Output vlogging file" OFF)
if (SUPPORT_VERIFIER)
  message(STATUS "Output vlogging file")
  add_definitions(-DSUPPORT_VERIFIER=1)
endif()

option(CODEC_CAP "Codec capability check" ON)
option(MULTICHANNEL_BINAURALIZER "Enable multichannel binaural rendering" OFF)
option(HOA_BINAURALIZER "Enable HOA binaural rendering" OFF)

if (CMAKE_BUILD_TYPE)
  string(TOLOWER ${CMAKE_BUILD_TYPE} BUILD_TYPE)
  if ("${BUILD_TYPE}" STREQUAL "debug")
    add_definitions(-DIA_DBG)
    message(STATUS "Debug mode.")
  endif()
endif()

set(IAMF_BEAR_PATH "${EXTER_LIB_DIR}/binaural/libiamf2bear.so")
message(WARNING "bear path = ${IAMF_BEAR_PATH}")
if(MULTICHANNEL_BINAURALIZER)
  message(STATUS "Enable multichannel binaural rendering")
  add_library(iamf2bear SHARED IMPORTED)
  add_definitions(-DENABLE_MULTICHANNEL_TO_BINAURAL=1)

  if(EXISTS "${IAMF_BEAR_PATH}")
    set_target_properties(
            iamf2bear PROPERTIES
            IMPORTED_LOCATION "${IAMF_BEAR_PATH}"
    )
  else()
    message(WARNING "The library 'libiamf2bear.so' does not exist at ${IAMF_BEAR_PATH}.")
  endif()
endif()

set(IAMF_RESONANCE_PATH "${EXTER_LIB_DIR}/binaural/libiamf2resonance.so")
message(WARNING "resonance path = ${IAMF_RESONANCE_PATH}")
if(HOA_BINAURALIZER)
  message(STATUS "Enable HOA binaural rendering")
  add_library(iamf2resonance SHARED IMPORTED)
  add_definitions(-DENABLE_HOA_TO_BINAURAL=1)

  if(EXISTS "${IAMF_RESONANCE_PATH}")
    set_target_properties(
            iamf2resonance PROPERTIES
            IMPORTED_LOCATION "${IAMF_RESONANCE_PATH}"
    )
  else()
    message(WARNING "The library 'libiamf2resonance.so' does not exist at ${IAMF_RESONANCE_PATH}.")
  endif()
endif()

add_subdirectory(opus)
add_subdirectory(flac)

add_library(libopus STATIC IMPORTED)
add_library(libflac STATIC IMPORTED)


set(OPUS_PATH "${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libopus.a")
set(FLAC_PATH "${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libFLAC.a")
set(AAC_PATH "${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libaac.a")

if(EXISTS "${OPUS_PATH}")
  set_target_properties(
          libopus PROPERTIES
          IMPORTED_LOCATION "${OPUS_PATH}"
  )
  add_definitions(-DCONFIG_OPUS_CODEC)
  aux_source_directory (src/iamf_dec/opus DIR_IAMF_DEC_OPUS)
else()
  message(WARNING "The library 'libopus.a' does not exist at ${OPUS_PATH}.")
endif()

if(EXISTS "${FLAC_PATH}")
  set_target_properties(
          libflac PROPERTIES
          IMPORTED_LOCATION "${FLAC_PATH}"
  )
  add_definitions(-DCONFIG_FLAC_CODEC)
  aux_source_directory (src/iamf_dec/flac DIR_IAMF_DEC_FLAC)
else()
  message(WARNING "The library 'libFLAC.a' does not exist at ${FLAC_PATH}.")
endif()

if(EXISTS "${AAC_PATH}")
  add_library(libaac STATIC IMPORTED)
  message(WARNING "The library 'libaac.a'  exist at ${AAC_PATH}.")
  set_target_properties(
          libaac PROPERTIES
          IMPORTED_LOCATION "${AAC_PATH}"
  )
  add_definitions(-DCONFIG_AAC_CODEC)
  aux_source_directory (src/iamf_dec/aac DIR_IAMF_DEC_AAC)
else()
  message(WARNING "The library 'libaac.a' does not exist at ${AAC_PATH}.")
endif()
aux_source_directory (src/iamf_dec/pcm DIR_IAMF_DEC_PCM)

include_directories (include)
aux_source_directory (dep_external/src/wav DIR_DEP_EXTERNAL_WAV)
aux_source_directory (src/common DIR_IAMF_COMMON)
aux_source_directory (src/iamf_dec DIR_IAMF_DEC)

#get_target_property(OPUS_LOCATION libopus IMPORTED_LOCATION)
#get_target_property(FLAC_LOCATION libopus IMPORTED_LOCATION)
#get_target_property(AAC_LOCATION libopus IMPORTED_LOCATION)

include_directories(
        ${CODEC_INCLUDE_DIR}
        ${EXTER_INCLUDE_DIR}
        ${PROJECT_SOURCE_DIR}/src/common
        ${PROJECT_SOURCE_DIR}/src/iamf_dec
        ${PROJECT_SOURCE_DIR}/src/iamf_dec/opus
        ${PROJECT_SOURCE_DIR}/src/iamf_dec/aac
        ${PROJECT_SOURCE_DIR}/src/iamf_dec/flac
)
link_directories (
        ${CODEC_LIB_DIR}
)

if(MULTICHANNEL_BINAURALIZER OR HOA_BINAURALIZER)
  link_directories(
          ${EXTER_LIB_DIR}/binaural
  )
endif()

if(BUILD_SHARED_LIBS)
  add_library(${PROJECT_NAME} SHARED ${DIR_DEP_EXTERNAL_WAV} ${DIR_IAMF_COMMON}
          ${DIR_IAMF_DEC_OPUS} ${DIR_IAMF_DEC_AAC} ${DIR_IAMF_DEC_FLAC} ${DIR_IAMF_DEC_PCM} ${DIR_IAMF_DEC})
  if(EXISTS "${OPUS_PATH}")
    target_link_libraries (${PROJECT_NAME} libopus)
  endif()
  if(EXISTS "${FLAC_PATH}")
    target_link_libraries (${PROJECT_NAME} libflac)
  endif()
  if(EXISTS "${AAC_PATH}")
    target_link_libraries (${PROJECT_NAME} libaac)
  endif()
  if(MULTICHANNEL_BINAURALIZER)
    target_link_libraries (${PROJECT_NAME} iamf2bear)
  endif()
  if(HOA_BINAURALIZER)
    target_link_libraries (${PROJECT_NAME} iamf2resonance)
  endif()
  target_link_libraries (${PROJECT_NAME} m)
  #  target_link_libraries (${PROJECT_NAME} ${IAMF_BEAR_PATH})
else()
  add_library(${PROJECT_NAME} STATIC ${DIR_DEP_EXTERNAL_WAV} ${DIR_IAMF_COMMON}
          ${DIR_IAMF_DEC_OPUS} ${DIR_IAMF_DEC_AAC} ${DIR_IAMF_DEC_PCM}
          ${DIR_IAMF_DEC_FLAC} ${DIR_IAMF_DEC})
endif()

set(IAMF_PUBLIC_HEADER
        ${CMAKE_CURRENT_SOURCE_DIR}/include/IAMF_decoder.h
        ${CMAKE_CURRENT_SOURCE_DIR}/include/IAMF_defines.h
        ${CMAKE_CURRENT_SOURCE_DIR}/include/vlogging_tool_sr.h
)
set_target_properties(${PROJECT_NAME} PROPERTIES PUBLIC_HEADER "${IAMF_PUBLIC_HEADER}")


set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix ${CMAKE_INSTALL_PREFIX})
set(libdir ${CMAKE_INSTALL_PREFIX}/lib)
set(includedir ${CMAKE_INSTALL_PREFIX}/include)
configure_file(iamf.pc.in iamf.pc)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/iamf.pc
        DESTINATION ${CMAKE_INSTALL_PREFIX}/lib/pkgconfig)

if(BUILD_SHARED_LIBS)
  install(TARGETS ${PROJECT_NAME}
          LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
          PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_PREFIX}/include/iamf)
else()
  install(TARGETS ${PROJECT_NAME}
          ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
          PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_PREFIX}/include/iamf)
endif()

if(MULTICHANNEL_BINAURALIZER OR HOA_BINAURALIZER)
  install(DIRECTORY ${EXTER_LIB_DIR}/binaural
          DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)
endif()

set_target_properties(
        ${PROJECT_NAME} PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY ${CMAKE_INSTALL_PREFIX}/${ANDROID_ABI}
)

