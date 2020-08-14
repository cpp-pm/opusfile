cmake_minimum_required(VERSION 3.4)

option(BUILD_OPUSURL "Build opusurl library" ON)

project(opusfile C)

if(MSVC)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4267 /wd4244 /wd4090")
endif()

find_path(OGG_INCLUDE_DIR ogg/ogg.h)
find_library(OGG_LIBRARY ogg)

find_path(OPUS_INCLUDE_DIR opus.h PATH_SUFFIXES opus)
find_library(OPUS_LIBRARY opus)

add_library(opusfile
    src/info.c
    src/internal.c
    src/opusfile.c
    src/stream.c)

target_include_directories(opusfile PRIVATE include PUBLIC ${OGG_INCLUDE_DIR} ${OPUS_INCLUDE_DIR})
target_link_libraries(opusfile PRIVATE ${OGG_LIBRARY} ${OPUS_LIBRARY})

install(TARGETS opusfile
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib)

if(BUILD_OPUSURL)
    find_package(OpenSSL MODULE REQUIRED)

    set(opusurl_src
        src/http.c
        src/internal.c)

    if(MSVC)
    list(APPEND opusurl_src
        src/wincerts.c)
    endif()

    add_library(opusurl ${opusurl_src})

    target_compile_definitions(opusurl PRIVATE OP_ENABLE_HTTP)
    target_include_directories(opusurl PRIVATE include)
    target_link_libraries(opusurl PRIVATE opusfile OpenSSL::SSL)

    if(MSVC)
    target_link_libraries(opusurl PRIVATE ws2_32.lib crypt32.lib)
    endif()

    install(TARGETS opusurl
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)
endif()

install(FILES include/opusfile.h
    DESTINATION include/opus)
