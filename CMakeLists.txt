# media-server库构建配置
cmake_minimum_required(VERSION 3.10)

project(media-server C CXX)

# SDK依赖
set(SDK_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/../sdk)
include_directories(${SDK_ROOT}/include)

# avcodec依赖
set(AVCODEC_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/../avcodec)
include_directories(${AVCODEC_ROOT}/avcodec/include)
include_directories(${AVCODEC_ROOT}/avbsf/include)

# media-server内部库依赖
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/libflv/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/libmkv/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/libmpeg/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/libmov/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/librtp/include)

# 辅助宏：创建库，自动检测src或source目录
macro(add_media_library lib_name)
    # 尝试source目录
    file(GLOB_RECURSE LIB_SOURCES
        ${CMAKE_CURRENT_SOURCE_DIR}/${lib_name}/source/*.c
        ${CMAKE_CURRENT_SOURCE_DIR}/${lib_name}/source/*.cpp
    )

    # 如果source目录为空，尝试src目录
    if(NOT LIB_SOURCES)
        file(GLOB_RECURSE LIB_SOURCES
            ${CMAKE_CURRENT_SOURCE_DIR}/${lib_name}/src/*.c
            ${CMAKE_CURRENT_SOURCE_DIR}/${lib_name}/src/*.cpp
        )
    endif()

    if(LIB_SOURCES)
        add_library(${lib_name} STATIC ${LIB_SOURCES})

        target_include_directories(${lib_name} PUBLIC
            ${CMAKE_CURRENT_SOURCE_DIR}/${lib_name}/include
        )

        # 提取库名（去掉lib前缀）
        string(REPLACE "lib" "" short_name ${lib_name})

        set_target_properties(${lib_name} PROPERTIES
            OUTPUT_NAME "${short_name}"
            ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
        )

        # 添加编译选项
        if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
            target_compile_options(${lib_name} PRIVATE -Wall)
        endif()

        list(LENGTH LIB_SOURCES source_count)
        message(STATUS "  ${lib_name}: ${CMAKE_BINARY_DIR}/lib/lib${short_name}.a (${source_count} sources)")
    else()
        message(STATUS "  ${lib_name}: SKIPPED (no sources found)")
    endif()

    unset(LIB_SOURCES)
endmacro()

message(STATUS "Configuring media-server libraries:")

# 创建所有库（根据选项）
# 不常用的库默认不编译
if(BUILD_LIBDASH)
    add_media_library(libdash)
else()
    message(STATUS "  libdash: DISABLED (use -DBUILD_LIBDASH=ON to enable)")
endif()

add_media_library(libflv)

if(BUILD_LIBHLS)
    add_media_library(libhls)
else()
    message(STATUS "  libhls: DISABLED (use -DBUILD_LIBHLS=ON to enable)")
endif()

add_media_library(libmkv)
add_media_library(libmov)
add_media_library(libmpeg)

if(BUILD_LIBRTMP)
    # librtmp needs source dir and aio dir
    file(GLOB_RECURSE LIBRTMP_SOURCES
        ${CMAKE_CURRENT_SOURCE_DIR}/librtmp/source/*.c
        ${CMAKE_CURRENT_SOURCE_DIR}/librtmp/source/*.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/librtmp/aio/*.c
        ${CMAKE_CURRENT_SOURCE_DIR}/librtmp/aio/*.cpp
    )
    if(LIBRTMP_SOURCES)
        add_library(librtmp STATIC ${LIBRTMP_SOURCES})
        target_include_directories(librtmp PUBLIC
            ${CMAKE_CURRENT_SOURCE_DIR}/librtmp/include
            ${CMAKE_CURRENT_SOURCE_DIR}/librtmp/aio
        )
        set_target_properties(librtmp PROPERTIES
            OUTPUT_NAME "rtmp"
            ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
        )
        if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
            target_compile_options(librtmp PRIVATE -Wall)
        endif()
        list(LENGTH LIBRTMP_SOURCES source_count)
        message(STATUS "  librtmp: ${CMAKE_BINARY_DIR}/lib/librtmp.a (${source_count} sources)")
    else()
        message(STATUS "  librtmp: SKIPPED (no sources found)")
    endif()
else()
    message(STATUS "  librtmp: DISABLED (use -DBUILD_LIBRTMP=ON to enable)")
endif()

# librtp需要包含source、payload和rtpext目录
file(GLOB_RECURSE LIBRTP_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/librtp/source/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/librtp/source/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/librtp/payload/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/librtp/payload/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/librtp/rtpext/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/librtp/rtpext/*.cpp
)
if(LIBRTP_SOURCES)
    add_library(librtp STATIC ${LIBRTP_SOURCES})
    target_include_directories(librtp PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/librtp/include
    )
    set_target_properties(librtp PROPERTIES
        OUTPUT_NAME "rtp"
        ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    )
    if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
        target_compile_options(librtp PRIVATE -Wall)
    endif()
    list(LENGTH LIBRTP_SOURCES source_count)
    message(STATUS "  librtp: ${CMAKE_BINARY_DIR}/lib/librtp.a (${source_count} sources)")
else()
    message(STATUS "  librtp: SKIPPED (no sources found)")
endif()
add_media_library(librtsp)

if(BUILD_LIBSIP)
    add_media_library(libsip)
else()
    message(STATUS "  libsip: DISABLED (use -DBUILD_LIBSIP=ON to enable)")
endif()

message(STATUS "media-server libraries configured")
