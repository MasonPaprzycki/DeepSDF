cmake_minimum_required(VERSION 3.10)
cmake_policy(SET CMP0002 NEW)
project(DeepSDF LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)
include(ExternalProject)

# --- Silence Eigen and NANOFLANN stuff ---
set(EIGEN_BUILD_TESTING OFF CACHE BOOL "" FORCE)
set(EIGEN_BUILD_DOC OFF CACHE BOOL "" FORCE)
set(EIGEN_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)

# --- CLI11 ---
FetchContent_Declare(
  CLI11
  GIT_REPOSITORY https://github.com/CLIUtils/CLI11.git
  GIT_TAG v2.4.2
)
FetchContent_MakeAvailable(CLI11)

# --- Eigen ---
FetchContent_Declare(
  eigen
  GIT_REPOSITORY https://gitlab.com/libeigen/eigen.git
  GIT_TAG 3.4.0
)
FetchContent_Populate(eigen)
add_subdirectory(${eigen_SOURCE_DIR} ${eigen_BINARY_DIR})


# --- ZLIB ---
ExternalProject_Add(
    zlib_proj
    PREFIX ${CMAKE_BINARY_DIR}/zlib
    GIT_REPOSITORY https://github.com/madler/zlib.git
    GIT_TAG v1.2.13
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/zlib-install
        -DCMAKE_POSITION_INDEPENDENT_CODE=ON
    UPDATE_COMMAND ""
)

set(ZLIB_INCLUDE_DIR "${CMAKE_BINARY_DIR}/zlib-install/include" CACHE PATH "ZLIB include")
set(ZLIB_LIBRARY "${CMAKE_BINARY_DIR}/zlib-install/lib/zlibstatic.lib" CACHE FILEPATH "ZLIB lib")

# --- cnpy depends on zlib ---
add_subdirectory(third-party/zlib/cnpy)
add_dependencies(cnpy zlib_proj)



include(FetchContent)

FetchContent_Declare(
    nanoflann
    GIT_REPOSITORY https://github.com/YourUsername/nanoflann.git
    GIT_TAG v1.5.5  # or your branch/commit
)

FetchContent_GetProperties(nanoflann)
if(NOT nanoflann_POPULATED)
    # Populate the repository WITHOUT building it
    FetchContent_Populate(
        nanoflann
        # Explicitly disable any attempt to build
        SOURCE_DIR ${nanoflann_SOURCE_DIR}
        BINARY_DIR ${nanoflann_BINARY_DIR}
    )
endif()

# Create a header-only INTERFACE target
add_library(nanoflann INTERFACE)

# Set the include path to where your headers actually live
target_include_directories(nanoflann INTERFACE ${nanoflann_SOURCE_DIR})


# --- Pangolin via ExternalProject ---
option(WITH_PANGOLIN "Build with Pangolin (OpenGL visualization)" ON)
if(WITH_PANGOLIN)
    message(STATUS "Building with Pangolin support via ExternalProject")

    ExternalProject_Add(
        pangolin_proj
        PREFIX ${CMAKE_BINARY_DIR}/pangolin
        GIT_REPOSITORY https://github.com/stevenlovegrove/Pangolin.git
        GIT_TAG master
        CMAKE_ARGS
            -DPANGOLIN_USE_GLEW=ON
            -DPANGOLIN_BUILD_EXAMPLES=OFF
            -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/pangolin-install
        UPDATE_COMMAND ""
    )

    include_directories(${CMAKE_BINARY_DIR}/pangolin-install/include)
    link_directories(${CMAKE_BINARY_DIR}/pangolin-install/lib)

    set(PANGOLIN_LIB pangolin)
    set(PANGOLIN_DEP pangolin_proj)
endif()

# --- Shared sources ---
set(SOURCES
    src/ShaderProgram.cpp
    src/Utils.cpp
)

# --- deepSDF executable ---
file(GLOB SRC_FILES src/*.cpp)
add_executable(deepSDF ${SRC_FILES})
target_link_libraries(deepSDF PRIVATE CLI11::CLI11 Eigen3::Eigen nanoflann cnpy)
if(WITH_PANGOLIN)
    add_dependencies(deepSDF ${PANGOLIN_DEP})
    target_link_libraries(deepSDF PRIVATE ${PANGOLIN_LIB})
endif()
set_target_properties(deepSDF PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin")

# --- PreprocessMesh ---
add_executable(PreprocessMesh src/PreprocessMesh.cpp ${SOURCES})
target_include_directories(PreprocessMesh PRIVATE third-party/zlib/cnpy)
target_link_libraries(PreprocessMesh PRIVATE CLI11::CLI11 Eigen3::Eigen cnpy nanoflann)
if(WITH_PANGOLIN)
    add_dependencies(PreprocessMesh ${PANGOLIN_DEP})
    target_link_libraries(PreprocessMesh PRIVATE ${PANGOLIN_LIB})
endif()
set_target_properties(PreprocessMesh PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin")

# --- SampleVisibleMeshSurface ---
add_executable(SampleVisibleMeshSurface src/SampleVisibleMeshSurface.cpp ${SOURCES})
target_include_directories(SampleVisibleMeshSurface PRIVATE third-party/zlib/cnpy)
target_link_libraries(SampleVisibleMeshSurface PRIVATE CLI11::CLI11 Eigen3::Eigen nanoflann cnpy)
if(WITH_PANGOLIN)
    add_dependencies(SampleVisibleMeshSurface ${PANGOLIN_DEP})
    target_link_libraries(SampleVisibleMeshSurface PRIVATE ${PANGOLIN_LIB})
endif()
set_target_properties(SampleVisibleMeshSurface PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin")
