cmake_minimum_required(VERSION 3.15)

option(CUDALIB "Enable CUDA training" OFF)

set(PRECISION "BF16" CACHE STRING "Precision Settings")

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Always export compile_commands.json for lsp like clangd.
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# We don't support this compiler.
if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  message(FATAL_ERROR "This compiler is not supported")
endif()

if (CUDALIB)
  project(llm.c LANGUAGES C CXX CUDA)
else()
  project(llm.c LANGUAGES C)
endif()

# Put binaries and libraries in the same location.
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

# Release by default if not specified.
if (NOT EXISTS ${CMAKE_BINARY_DIR}/CMakeCache.txt)
  if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
  endif()
endif()

add_library(train_gpt2_cpu train_gpt2.c)
target_include_directories(train_gpt2_cpu PUBLIC ${CMAKE_CURRENT_SOURCE_DIR} ${PROJECT_SOURCE_DIR}/llmc)
target_link_libraries(train_gpt2_cpu PRIVATE m)
target_compile_definitions(train_gpt2_cpu PRIVATE -DLLMC_LIB=1)
if (NO_OMP)
  message(STATUS "OpenMP is manually disabled")
else()
  find_package(OpenMP)
  if (OpenMP_FOUND)
    message(STATUS "✓ OpenMP found")
    target_link_libraries(train_gpt2_cpu PRIVATE OpenMP::OpenMP_C)
  else()
    message(STATUS "✗ OpenMP not found")
  endif()
endif()
target_compile_options(train_gpt2_cpu PRIVATE -Ofast -Wno-unused-result -Wno-ignored-pragmas -Wno-unknown-attributes -march=native)

# Training GPT2 with CUDA.
if (CUDALIB)
  set_source_files_properties(llmc/cudnn_att.cpp PROPERTIES LANGUAGE CUDA)
  add_library(train_gpt2_cuda train_gpt2.cu llmc/cudnn_att.cpp)
  target_include_directories(train_gpt2_cuda PUBLIC ${CMAKE_CURRENT_SOURCE_DIR} ${PROJECT_SOURCE_DIR}/llmc)
  target_compile_options(train_gpt2_cuda PRIVATE -O3 -t=0 --use_fast_math)
  target_compile_definitions(train_gpt2_cuda PRIVATE -DLLMC_LIB=1)
  set_target_properties(train_gpt2_cuda PROPERTIES CXX_STANDARD 17)
  set_target_properties(train_gpt2_cuda PROPERTIES CUDA_ARCHITECTURES "72;80")

  if (PRECISION STREQUAL "FP32")
    target_compile_definitions(train_gpt2_cuda PRIVATE -DENABLE_FP32)
  elseif(PRECISION STREQUAL "FP16")
    target_compile_definitions(train_gpt2_cuda PRIVATE -DENABLE_FP16)
  else()
    target_compile_definitions(train_gpt2_cuda PRIVATE -DENABLE_BF16)
  endif()

  set(CUDNN_FRONTEND_BUILD_SAMPLES OFF)
  set(CUDNN_FRONTEND_BUILD_UNIT_TESTS OFF)
  message(STATUS "Fetching cudnn-frontend")
  include(FetchContent)
  FetchContent_Declare(
    cf
    URL https://github.com/NVIDIA/cudnn-frontend/archive/refs/tags/v1.6.1.tar.gz
    URL_HASH MD5=c131914d8007318ec7b5b5f792458cb4
  )
  FetchContent_MakeAvailable(cf)
  FetchContent_GetProperties(cf)
  target_include_directories(train_gpt2_cuda PRIVATE ${cf_SOURCE_DIR}/include)
  target_compile_definitions(train_gpt2_cuda PRIVATE -DENABLE_CUDNN)
  target_link_libraries(train_gpt2_cuda PRIVATE cudnn)

  find_package(CUDAToolkit REQUIRED)
  target_link_libraries(train_gpt2_cuda PRIVATE CUDA::cublas CUDA::cublasLt CUDA::cudart CUDA::nvrtc)

  if (NO_USE_MPI)
    message(STATUS "→ MPI is manually disabled")
  else()
    find_package(MPI)
    if (MPI_FOUND)
      message(STATUS "✓ MPI found")
      target_compile_definitions(train_gpt2_cuda PRIVATE -DUSE_MPI)
      target_link_libraries(train_gpt2_cuda PRIVATE MPI::MPI_C)
    else()
        message(STATUS "✗ MPI not found")
    endif()
  endif()
endif()

