cmake_minimum_required(VERSION 3.10)
set(CMAKE_CXX_STANDARD 17)
find_package(Git)
execute_process(
  COMMAND ${GIT_EXECUTABLE} log -1 --format=%h
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_COMMIT_HASH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
add_definitions("-DGIT_COMMIT_HASH=\"${GIT_COMMIT_HASH}\"")

project(chia-plotter C CXX ASM)

if(NOT CHIA_PLOT_BUILD_TESTS)
    set(CHIA_PLOT_BUILD_TESTS true)
endif()


set(TARGET_ARCH ${CMAKE_HOST_SYSTEM_PROCESSOR})
message(STATUS "Architecture: ${TARGET_ARCH}")
set(ENV{RELIC_MAIN} "1")

if (MSVC)

    add_definitions(-DCHIA_PLOT_BUILD_INFO="${CHIA_PLOT_BUILD_INFO}")
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
	add_definitions(-DSODIUM_STATIC=TRUE)

    add_definitions(-D_CRT_SECURE_NO_WARNINGS)

	include_directories(
		${sodium_INCLUDE_DIR}
		${GMP_INCLUDE_DIR}
	)	

else()
	find_package(Threads REQUIRED)
	
	if(APPLE)
		find_package(sodium REQUIRED)
	endif()

	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-unused-function")
endif()

add_subdirectory(lib/bls-signatures)

add_subdirectory(lib/libbech32)

set(BLAKE3_PATH lib/BLAKE3/c)
set(FSE_PATH lib/FSE/lib)

include_directories(
	lib
	include
	lib/bls-signatures/src
	lib/libbech32/include/libbech32
	${BLAKE3_PATH}
	${CMAKE_BINARY_DIR}/_deps/relic-src/include
	${CMAKE_BINARY_DIR}/_deps/relic-build/include
)

#add_definitions(-DBLAKE3_NO_SSE2=true)
#add_definitions(-DBLAKE3_NO_SSE41=true)
#add_definitions(-DBLAKE3_NO_AVX2=true)
#add_definitions(-DBLAKE3_NO_AVX512=true)

IF (WIN32)
set(BLAKE3_SRC
    ${BLAKE3_PATH}/blake3.c
    ${BLAKE3_PATH}/blake3_portable.c
    ${BLAKE3_PATH}/blake3_dispatch.c

    ${BLAKE3_PATH}/blake3_sse2.c
    ${BLAKE3_PATH}/blake3_sse41.c
    ${BLAKE3_PATH}/blake3_avx2.c
    ${BLAKE3_PATH}/blake3_avx512.c
)
ELSEIF(TARGET_ARCH STREQUAL "x86_64")
set(BLAKE3_SRC
    ${BLAKE3_PATH}/blake3.c
    ${BLAKE3_PATH}/blake3_portable.c
    ${BLAKE3_PATH}/blake3_dispatch.c
    ${BLAKE3_PATH}/blake3_avx2_x86-64_unix.S
    ${BLAKE3_PATH}/blake3_avx512_x86-64_unix.S
    ${BLAKE3_PATH}/blake3_sse41_x86-64_unix.S
    ${BLAKE3_PATH}/blake3_sse2_x86-64_unix.S
)
ELSE()
set(BLAKE3_SRC
    ${BLAKE3_PATH}/blake3.c
    ${BLAKE3_PATH}/blake3_portable.c
    ${BLAKE3_PATH}/blake3_dispatch.c
)
ENDIF()

set(FSE_SRC
    ${FSE_PATH}/fse_compress.c
    ${FSE_PATH}/fse_decompress.c
    ${FSE_PATH}/entropy_common.c
    ${FSE_PATH}/hist.c
)

add_library(blake3 STATIC ${BLAKE3_SRC})

add_library(fse STATIC ${FSE_SRC})

add_library(chia_plotter STATIC
	lib/chacha8.c
	src/settings.cpp
)

if (MSVC)
	target_link_libraries(chia_plotter blake3 fse bech32)
else()
	target_link_libraries(chia_plotter blake3 fse bech32 Threads::Threads)
endif()

if(CHIA_PLOT_BUILD_TESTS)
	add_executable(test_copy test/test_copy.cpp)
	add_executable(test_disk_sort test/test_disk_sort.cpp)

	add_executable(test_phase_1 test/test_phase_1.cpp)
	add_executable(test_phase_2 test/test_phase_2.cpp)
	add_executable(test_phase_3 test/test_phase_3.cpp)
	add_executable(test_phase_4 test/test_phase_4.cpp)

	add_executable(check_phase_1 test/check_phase_1.cpp)
endif()

add_executable(chia_plot src/chia_plot.cpp)

if (MSVC)

	add_library(uint128 STATIC uint128_t/uint128_t.cpp)
	target_include_directories(uint128 PUBLIC uint128_t)

	if(CHIA_PLOT_BUILD_TESTS)
		target_link_libraries(test_copy chia_plotter)
		target_link_libraries(test_disk_sort chia_plotter uint128)
		target_link_libraries(test_phase_1 chia_plotter uint128)
		target_link_libraries(test_phase_2 chia_plotter uint128)
		target_link_libraries(test_phase_3 chia_plotter uint128)
		target_link_libraries(test_phase_4 chia_plotter uint128)
		target_link_libraries(check_phase_1 chia_plotter uint128)
	endif()

	target_link_libraries(chia_plot chia_plotter bls uint128)
	set_target_properties(chia_plot PROPERTIES LINK_OPTIONS -NODEFAULTLIB:MSVCRT)

else()

	if(CHIA_PLOT_BUILD_TESTS)	
		target_link_libraries(test_copy chia_plotter)
		target_link_libraries(test_disk_sort chia_plotter)

		target_link_libraries(test_phase_1 chia_plotter)
		target_link_libraries(test_phase_2 chia_plotter)
		target_link_libraries(test_phase_3 chia_plotter)
		target_link_libraries(test_phase_4 chia_plotter)

		target_link_libraries(check_phase_1 chia_plotter)
	endif()	

	target_link_libraries(chia_plot chia_plotter bls sodium)

endif()

