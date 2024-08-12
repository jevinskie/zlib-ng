cmake_minimum_required(VERSION 3.12)

include(FetchContent)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)

enable_language(CXX)

# Search for Google benchmark package
find_package(benchmark QUIET)
if(NOT benchmark_FOUND)
    # Fetch google benchmark source code from official repository
    set(BENCHMARK_ENABLE_TESTING OFF)

    # Allow specifying alternative Google benchmark repository
    if(NOT DEFINED GBENCHMARK_REPOSITORY)
        set(GBENCHMARK_REPOSITORY https://github.com/google/benchmark.git)
    endif()
    if(NOT DEFINED GBENCHMARK_TAG)
        set(GBENCHMARK_TAG v1.7.1)
    endif()

    FetchContent_Declare(benchmark
        GIT_REPOSITORY ${GBENCHMARK_REPOSITORY}
        GIT_TAG ${GBENCHMARK_TAG})

    FetchContent_GetProperties(benchmark)
    if(NOT benchmark_POPULATED)
        FetchContent_Populate(benchmark)
        add_subdirectory(${benchmark_SOURCE_DIR} ${benchmark_BINARY_DIR} EXCLUDE_FROM_ALL)
    endif()
endif()

add_executable(benchmark_zlib
    benchmark_adler32.cc
    benchmark_adler32_copy.cc
    benchmark_compare256.cc
    benchmark_compare256_rle.cc
    benchmark_compress.cc
    benchmark_crc32.cc
    benchmark_main.cc
    benchmark_slidehash.cc
    )

target_compile_definitions(benchmark_zlib PRIVATE -DBENCHMARK_STATIC_DEFINE)
target_include_directories(benchmark_zlib PRIVATE
    ${PROJECT_SOURCE_DIR}
    ${PROJECT_BINARY_DIR}
    ${benchmark_SOURCE_DIR}/benchmark/include)

target_link_libraries(benchmark_zlib zlibstatic benchmark::benchmark)
if(WIN32)
    target_link_libraries(benchmark_zlib shlwapi)
endif()

add_test(NAME benchmark_zlib
    COMMAND ${CMAKE_CROSSCOMPILING_EMULATOR} $<TARGET_FILE:benchmark_zlib>)

if(WITH_BENCHMARK_APPS)
    if(NOT ZLIB_COMPAT)
        message(FATAL_ERROR "Building with WITH_BENCHMARK_APPS requires ZLIB_COMPAT=ON")
    endif()
    option(BUILD_ALT_BENCH "Link against alternative zlib implementation" OFF)

    # Search for libpng package
    find_package(PNG)
    if(PNG_FOUND)
        if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.20)
            cmake_path(REPLACE_EXTENSION PNG_LIBRARY LAST_ONLY "${CMAKE_STATIC_LIBRARY_SUFFIX}")
        else()
            string(FIND "${PNG_LIBRARY}" "${CMAKE_SHARED_LIBRARY_SUFFIX}" PNG_SHARED_IDX REVERSE)
            string(FIND "${PNG_LIBRARY}" "${CMAKE_STATIC_LIBRARY_SUFFIX}" PNG_STATIC_IDX REVERSE)
            if (${PNG_SHARED_IDX} EQUAL -1)
                if (${PNG_STATIC_IDX} GREATER 0)
                    message(STATUS "find_package(PNG)'s found PNG_LIBRARY: \"${PNG_LIBRARY}\" is already static.")
                endif()
            else()
                string(SUBSTRING "${PNG_LIBRARY}" 0 ${PNG_SHARED_IDX} PNG_LIBRARY_PREFIX)
                string(CONCAT PNG_LIBRARY "${PNG_LIBRARY_PREFIX}" "${CMAKE_STATIC_LIBRARY_SUFFIX}") 
            endif()
        endif()
        message(STATUS "Checking if find_package(PNG) PNG_LIBRARY: \"${PNG_LIBRARY}\" exists.")
        if(EXISTS "${PNG_LIBRARY}")
            set(PNG_FOUND TRUE)
            set(PNG_LIB "${PNG_LIBRARY}")
            message(STATUS "find_package(PNG) PNG_LIB a.k.a. PNG_LIBRARY: found at \"${PNG_LIB}\"")
        else()
            message(STATUS "find_package(PNG) PNG_LIBRARY: \"${PNG_LIBRARY}\" file doesn't exist")
            set(PNG_FOUND FALSE)
        endif()
    endif()

    if(NOT PNG_FOUND)
        find_package(PkgConfig)
        if(PKG_CONFIG_FOUND)
            pkg_check_modules(PNG libpng)
            set(ZLIB_NG_PNG_LIB_FILE "${PNG_LIBRARIES}{CMAKE_STATIC_LIBRARY_SUFFIX}")
            if(NOT PNG_FOUND)
                message(STATUS "Couldn't find libpng using pkg-config for WITH_BENCHMARK_APPS")
            else()
                # if(FALSE)
                if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.20)
                    cmake_path(APPEND PNG_LIB "${PNG_LIBRARY_DIRS}" "lib${PNG_LIBRARIES}${CMAKE_STATIC_LIBRARY_SUFFIX}")
                    cmake_path(NORMAL_PATH PNG_LIB)
                else()
                    set(PNG_LIB "${PNG_LIBRARY_DIRS}/lib${PNG_LIBRARIES}${CMAKE_STATIC_LIBRARY_SUFFIX}" CACHE FILEPATH "libpng static library for WITH_BENCHMARK_APPS")
                endif()
            endif()
            if(NOT EXISTS "${PNG_LIB}")
                message(STATUS "Couldn't find libpng using pkg-config at \"${PNG_LIB}\" for WITH_BENCHMARK_APPS")
            else()
                set(PNG_FOUND TRUE)
                message(STATUS "Found libpng using pkg-config at \"${PNG_LIB}\" for WITH_BENCHMARK_APPS")
            endif()
        endif()
    endif()

    if(NOT PNG_FOUND)
        message(STATUS "Falling back to using FetchContent to get libpng for WITH_BENCHMARK_APPS")
        FetchContent_Declare(PNG
            GIT_REPOSITORY https://github.com/glennrp/libpng.git)

        FetchContent_GetProperties(PNG)
        if(NOT PNG_POPULATED)
            FetchContent_Populate(PNG)
            set(PNG_INCLUDE_DIR ${png_SOURCE_DIR})
            add_subdirectory(${png_SOURCE_DIR} ${png_BINARY_DIR})
            set(PNG_LIB libpng.a)
        endif()
    endif()

    set(BENCH_APP_SRCS
        benchmark_png_encode.cc
        benchmark_png_decode.cc
        benchmark_main.cc
    )

    add_executable(benchmark_zlib_apps ${BENCH_APP_SRCS})

    if(DEFINED BUILD_ALT_BENCH)
        if(NOT DEFINED ZLIB_ALT_LIB AND NOT ZLIB_COMPAT)
            set(ZLIB_ALT_LIB "libz.a" CACHE FILEPATH "Optional alternative zlib implementation (defaults to stock zlib)")
        endif()
        if(NOT EXISTS "${ZLIB_ALT_LIB}" OR ZLIB_COMPAT)
            unset(ZLIB_FOUND)
            unset(ZLIB_LIBRARIES)
            unset(ZLIB_INCLUDE_DIRECTORIES)
            find_package(PkgConfig)
            if(PKG_CONFIG_FOUND)
                pkg_check_modules(ZLIB_ALT zlib)
            endif()
            if(NOT ZLIB_ALT_FOUND)
                message(FATAL_ERROR "Couldn't find zlib for BUILD_ALT_BENCH")
            else()
                if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.20)
                    cmake_path(APPEND ZLIB_ALT_LIB "${ZLIB_ALT_LIBRARY_DIRS}" "lib${ZLIB_ALT_LIBRARIES}${CMAKE_STATIC_LIBRARY_SUFFIX}")
                    cmake_path(NORMAL_PATH ZLIB_ALT_LIB)
                else()
                    set(ZLIB_ALT_LIB "${ZLIB_ALT_LIBRARY_DIRS}/lib${ZLIB_ALT_LIBRARIES}${CMAKE_STATIC_LIBRARY_SUFFIX}" CACHE FILEPATH "Optional alternative zlib implementation (defaults to stock zlib)")
                endif()
            endif()
            if(NOT EXISTS "${ZLIB_ALT_LIB}")
                message(FATAL_ERROR "Couldn't find libz.a using pkg-config for BUILD_ALT_BENCH")
            else()
                message(STATUS "ZLIB_ALT_LIB: ${ZLIB_ALT_LIB}")
            endif()
        endif()
        add_executable(benchmark_zlib_apps_alt ${BENCH_APP_SRCS})
        target_link_libraries(benchmark_zlib_apps_alt ${PNG_LIB} ${ZLIB_ALT_LIB} benchmark::benchmark)
        target_compile_definitions(benchmark_zlib_apps_alt PRIVATE BUILD_ALT=1)
        target_include_directories(benchmark_zlib_apps_alt PRIVATE
            ${PROJECT_SOURCE_DIR}
            ${PROJECT_BINARY_DIR}
            ${PNG_INCLUDE_DIR}
            ${benchmark_SOURCE_DIR}/benchmark/include)
    endif()

    target_include_directories(benchmark_zlib_apps PRIVATE
        ${PROJECT_SOURCE_DIR}
        ${PROJECT_BINARY_DIR}
        ${PNG_INCLUDE_DIR}
        ${benchmark_SOURCE_DIR}/benchmark/include)

    # We need the static png library if we're statically linking to zlib,
    # otherwise it will resolve these things in the system provided dynamic
    # libraries (likely linked to stock zlib)
    target_link_libraries(benchmark_zlib_apps ${PNG_LIB} zlibstatic benchmark::benchmark)
endif()
