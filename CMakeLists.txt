cmake_minimum_required(VERSION 3.9 FATAL_ERROR)
# set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --allow-unsupported-compiler")
# set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} --allow-unsupported-compiler")
# set(CMAKE_CUDA_FLAGS_INIT "${CMAKE_CUDA_FLAGS_INIT} --allow-unsupported-compiler")
# set(CMAKE_CUDA_COMPILER_ID_REQUIRE_SUCCESS OFF)
# add_definitions(-D_ALLOW_COMPILER_AND_STL_VERSION_MISMATCH)

project(MRPNN LANGUAGES CXX CUDA)

set(ENV{http_proxy} "http://127.0.0.1:7890")
set(ENV{https_proxy} "http://127.0.0.1:7890")

# set(CUDAToolkit_ROOT $ENV{CUDA_PATH_V11_8})
find_package(CUDAToolkit)
link_libraries(CUDA::cudart)
include_directories(SYSTEM "${CUDAToolkit_INCLUDE_DIRS}")
message("${CUDAToolkit_INCLUDE_DIRS}")

IF (WIN32)

MESSAGE(STATUS "Is windows")
SET(CMAKE_CXX_FLAGS "/openmp /Ot")

ELSE ()

MESSAGE(STATUS "Is linux")
add_definitions(-DLINUX)

ENDIF ()

Option(GUI OFF)
Option(RTX30XX OFF)
Option(TENSORCORE OFF)
Option(CompileRPNN OFF)
Option(DISABLE_FAST_MATH OFF)
OPTION(Debug OFF)

include(FetchContent)
if (GUI)
    add_definitions(-DGUI)
    FetchContent_Declare(GLEW 
        SOURCE_DIR     ${CMAKE_SOURCE_DIR}/external/glew
        URL            https://sourceforge.net/projects/glew/files/glew/2.1.0/glew-2.1.0.zip
    )
    FetchContent_MakeAvailable(GLEW)
    add_subdirectory(${CMAKE_SOURCE_DIR}/external/glew/build/cmake)

    FetchContent_Declare(glfw3 
        SOURCE_DIR     ${CMAKE_SOURCE_DIR}/external/glfw3
        URL            https://github.com/glfw/glfw/releases/download/3.4/glfw-3.4.zip
    )
    FetchContent_MakeAvailable(glfw3)

    add_subdirectory(Imgui)
	INCLUDE_DIRECTORIES(Imgui_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/external/glew/include ${CMAKE_SOURCE_DIR}/external/glfw3/include)
endif()

if (CompileRPNN)
    add_definitions(-DCRPNN)
endif()

if (TENSORCORE)
    add_definitions(-DENBALE_TENSOR)
endif()
add_compile_options($<$<COMPILE_LANGUAGE:CUDA>:--extended-lambda>)
if (Debug)
add_compile_options($<$<COMPILE_LANGUAGE:CUDA>:-G>)
add_compile_options($<$<COMPILE_LANGUAGE:CUDA>:-lineinfo>)
endif()

file(GLOB CORE_HEADERS core/*.h core/*.hpp core/*.cuh)
file(GLOB CORE_SOURCES core/*.cpp core/*.cu)

source_group("Core Include" FILES ${CORE_HEADERS}) 
source_group("Core Source" FILES ${CORE_SOURCES})

LIST(REMOVE_ITEM CORE_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/core/gen.cu)

include_directories(${PROJECT_SOURCE_DIR}/core)
  
add_executable(DataGen ${CORE_HEADERS} ${CORE_SOURCES} dataGen.cu)
set_target_properties(DataGen PROPERTIES 
CUDA_SEPARABLE_COMPILATION ON
CXX_STANDARD_REQUIRED ON
CXX_STANDARD 17)
target_compile_definitions(DataGen PRIVATE _HAS_STD_BYTE=0)

add_executable(VolumeRender ${CMAKE_CURRENT_SOURCE_DIR}/core/gen.cu ${CORE_HEADERS} ${CORE_SOURCES} main.cu)
set_target_properties(VolumeRender PROPERTIES
CUDA_SEPARABLE_COMPILATION ON
CXX_STANDARD_REQUIRED ON
CXX_STANDARD 17)
target_compile_definitions(VolumeRender PRIVATE _HAS_STD_BYTE=0)

add_executable(Test ${CORE_HEADERS} ${CORE_SOURCES} test.cu)
set_target_properties(Test PROPERTIES
CUDA_SEPARABLE_COMPILATION ON
CXX_STANDARD_REQUIRED ON
CXX_STANDARD 17)
target_compile_definitions(Test PRIVATE _HAS_STD_BYTE=0)


if (DISABLE_FAST_MATH)
    if (RTX30XX)
    target_compile_options(VolumeRender PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-arch=sm_86 -maxrregcount=168>)
    target_compile_options(Test PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-arch=sm_86 -maxrregcount=168>)
    target_compile_options(DataGen PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-arch=sm_86 -maxrregcount=168>)
    else()
    target_compile_options(VolumeRender PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-arch=sm_75 -maxrregcount=168>)
    target_compile_options(Test PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-arch=sm_75 -maxrregcount=168>)
    target_compile_options(DataGen PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-arch=sm_75 -maxrregcount=168>)
    endif()
else()
    if (RTX30XX)
    target_compile_options(VolumeRender PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-use_fast_math -arch=sm_86 -maxrregcount=168>)
    target_compile_options(Test PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-use_fast_math -arch=sm_86 -maxrregcount=168>)
    target_compile_options(DataGen PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-use_fast_math -arch=sm_86 -maxrregcount=168>)
    else()
    target_compile_options(VolumeRender PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-use_fast_math -arch=sm_75 -maxrregcount=168>)
    target_compile_options(Test PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-use_fast_math -arch=sm_75 -maxrregcount=168>)
    target_compile_options(DataGen PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:-use_fast_math -arch=sm_75 -maxrregcount=168>)
    endif()
endif()

if (GUI)
    target_link_libraries(VolumeRender glew glfw Imgui) 
    add_dependencies(VolumeRender Imgui)
    target_link_libraries(Test glew glfw Imgui) 
    add_dependencies(Test Imgui)
    target_link_libraries(DataGen glew glfw Imgui) 
    add_dependencies(DataGen Imgui)
endif()