cmake_minimum_required(VERSION 3.10)
project(rnnoise LANGUAGES C)

set(MODEL_URL "https://media.xiph.org/rnnoise/models/rnnoise_data-0b50c45.tar.gz")

option(ENABLE_RUNTIME_CPU_DETECTION "Enable x86 Run-Time CPU Detection" ON)

set(RNNOISE_SOURCES
  src/denoise.c
  src/rnn.c
  src/pitch.c
  src/kiss_fft.c
  src/celt_lpc.c
  src/nnet.c
  src/nnet_default.c
  src/parse_lpcnet_weights.c
  src/rnnoise_data.c
  src/rnnoise_tables.c
)

# https://superuser.com/questions/305901/what-are-the-possible-values-of-processor-architecture/1441469#1441469
if(ENABLE_RUNTIME_CPU_DETECTION AND (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|i386|AMD64"))
  list(APPEND RNNOISE_SOURCES src/x86/x86_dnn_map.c src/x86/x86cpu.c src/x86/nnet_sse4_1.c src/x86/nnet_avx2.c)
endif()

file(DOWNLOAD "${MODEL_URL}" "${CMAKE_CURRENT_BINARY_DIR}/rnnoise_data.tar.gz"
  SHOW_PROGRESS
  STATUS DOWNLOAD_STATUS
)

list(GET DOWNLOAD_STATUS 0 STATUS_CODE)
if(NOT STATUS_CODE EQUAL 0)
  message(FATAL_ERROR "Download failed!")
endif()

file(ARCHIVE_EXTRACT INPUT "${CMAKE_CURRENT_BINARY_DIR}/rnnoise_data.tar.gz" DESTINATION "${CMAKE_CURRENT_SOURCE_DIR}")

add_library(rnnoise SHARED ${RNNOISE_SOURCES})

target_include_directories(rnnoise
  PUBLIC
    "${CMAKE_CURRENT_SOURCE_DIR}/include"
  PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/src"
)

if(ENABLE_RUNTIME_CPU_DETECTION AND (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|i386|AMD64"))
  set_source_files_properties(src/x86/nnet_sse4_1.c PROPERTIES
    COMPILE_OPTIONS "$<$<COMPILE_LANG_AND_ID:C,GNU,Clang,AppleClang>:-msse4.1>"
  )

  set_source_files_properties(src/x86/nnet_avx2.c PROPERTIES
    COMPILE_OPTIONS "$<$<COMPILE_LANG_AND_ID:C,GNU,Clang,AppleClang>:-mavx2;-mfma>"
  )
endif()
