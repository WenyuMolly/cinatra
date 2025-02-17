set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/tests)
set(project_name test_cinatra)
add_executable(${project_name}
        test_cinatra.cpp
        test_cinatra_websocket.cpp
        test_cmdline.cpp
        main.cpp
        )
target_compile_definitions(${project_name} PRIVATE ASYNC_SIMPLE_HAS_NOT_AIO INJECT_FOR_HTTP_CLIENT_TEST)
target_include_directories(${project_name} PRIVATE
        ${cinatra_SOURCE_DIR}/include
)

add_test(NAME ${project_name} COMMAND test_cinatra)

if (ENABLE_GZIP)
	target_link_libraries(${project_name} ${ZLIB_LIBRARIES})
endif()

# test_coro_file
option(ENABLE_FILE_IO_URING "enable io_uring" OFF)
if(ENABLE_FILE_IO_URING)
    message(STATUS "enable io_uring")
    add_definitions(-DENABLE_FILE_IO_URING)
    add_definitions(-DASIO_HAS_FILE)
    add_definitions(-DASIO_HAS_IO_URING)
endif()

if (UNIX)
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -pthread")
endif()

## manual import
include_directories(${cinatra_SOURCE_DIR}/include)

add_executable(test_corofile
        test_corofile.cpp
        )

if(ENABLE_FILE_IO_URING)
if (UNIX)
        target_link_libraries(test_corofile PRIVATE uring)
endif()
else()
        target_link_libraries(test_corofile PRIVATE)
endif()

add_test(NAME test_corofile COMMAND test_corofile)

add_executable(test_time_util
        test_time_util.cpp
        )
add_test(NAME test_time_util COMMAND test_time_util)

add_executable(test_coro_http_server test_coro_http_server.cpp)
add_test(NAME test_coro_http_server COMMAND test_coro_http_server)

option(CINATRA_ENABLE_SSL "Enable ssl support" OFF)
if (CINATRA_ENABLE_SSL)
    message(STATUS "Use SSL")
    find_package(OpenSSL REQUIRED)
    add_definitions(-DCINATRA_ENABLE_SSL)
    target_link_libraries(test_cinatra  OpenSSL::SSL OpenSSL::Crypto)
    target_link_libraries(test_corofile  PRIVATE OpenSSL::SSL OpenSSL::Crypto)
    target_link_libraries(test_coro_http_server  OpenSSL::SSL OpenSSL::Crypto)
endif ()

add_executable(test_http_parse
        test_http_parse.cpp
        )

if (ENABLE_SIMD STREQUAL "AARCH64")
	if (CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "aarch64")
		add_library(neon INTERFACE IMPORTED)
		target_compile_options(neon INTERFACE -march=armv8-a+fp+simd)
		target_link_libraries(test_http_parse neon)
	endif ()
elseif (ENABLE_SIMD STREQUAL "SSE42")
	if (CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "x86_64")
		add_library(sse4_2 INTERFACE IMPORTED)
		if(MSVC)
			target_compile_options(sse4_2 INTERFACE /arch:SSE4.2)
		else()
			target_compile_options(sse4_2 INTERFACE -msse4.2)
		endif()
		target_link_libraries(test_http_parse sse4_2)
	endif ()
elseif (ENABLE_SIMD STREQUAL "AVX2")
	if (CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "x86_64")
		add_library(avx2 INTERFACE IMPORTED)
		if(MSVC)
			target_compile_options(avx2 INTERFACE /arch:AVX2)
		else()
			target_compile_options(avx2 INTERFACE -mavx2)
		endif()
		target_link_libraries(test_http_parse avx2)
		set(CMAKE_CXX_FLAGS "-fpermissive")
	endif ()
endif ()

add_test(NAME test_http_parse COMMAND test_http_parse)
