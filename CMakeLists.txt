cmake_minimum_required(VERSION 3.4)

option(BUILD_OPUSURL "Build opusurl library" ON)

option(HUNTER_ENABLED "Enable Hunter package manager" OFF)

if(HUNTER_ENABLED)
    include(cmake/HunterGate.cmake)
    HunterGate(
        URL "https://github.com/cpp-pm/hunter/archive/v0.23.265.tar.gz"
        SHA1 "eae0bb5b7b8aa968e0ff87ec51be51b4255db110"
    )
endif()


project(opusfile VERSION 0.12 LANGUAGES C)

if(HUNTER_ENABLED)
    hunter_add_package(libogg)
    find_package(libogg CONFIG REQUIRED)
    set(OGG_LIBRARY libogg::ogg)
    get_target_property(OGG_INCLUDE_DIR libogg::ogg INTERFACE_INCLUDE_DIRECTORIES)

    hunter_add_package(Opus)
    find_package(Opus CONFIG REQUIRED)
    set(OPUS_LIBRARY Opus::opus)
    get_target_property(OPUS_INCLUDE_DIR Opus::opus INTERFACE_INCLUDE_DIRECTORIES)
    set(OPUS_INCLUDE_DIR ${OPUS_INCLUDE_DIR} ${OPUS_INCLUDE_DIR}/opus)
else()
    find_path(OGG_INCLUDE_DIR ogg/ogg.h)
    find_library(OGG_LIBRARY ogg)

    find_path(OPUS_INCLUDE_DIR opus.h PATH_SUFFIXES opus)
    find_library(OPUS_LIBRARY opus)
endif()

add_library(opusfile
    src/info.c
    src/internal.c
    src/opusfile.c
    src/stream.c)

if(MSVC)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4267 /wd4244 /wd4090")
endif()

target_include_directories(opusfile PRIVATE include PUBLIC ${OGG_INCLUDE_DIR} ${OPUS_INCLUDE_DIR})
target_link_libraries(opusfile PRIVATE ${OGG_LIBRARY} ${OPUS_LIBRARY})

set(INSTALL_TARGETS opusfile)

if(BUILD_OPUSURL)
    if(HUNTER_ENABLED)
        hunter_add_package(OpenSSL)
    endif()

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
    if(ANDROID)
        target_compile_definitions(opusurl PRIVATE OP_HAVE_CLOCK_GETTIME)
    endif()

    target_include_directories(opusurl PRIVATE include)
    target_link_libraries(opusurl PRIVATE opusfile OpenSSL::SSL)

    if(MSVC)
    target_link_libraries(opusurl PRIVATE ws2_32.lib crypt32.lib)
    endif()

    list(APPEND INSTALL_TARGETS opusurl)
endif()

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

set(config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

install(TARGETS ${INSTALL_TARGETS}
    EXPORT "${targets_export_name}"
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    FRAMEWORK DESTINATION ${CMAKE_INSTALL_LIBDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
    FILES ${CMAKE_SOURCE_DIR}/include/opusfile.h
    DESTINATION ${CMAKE_INSTALL_FULL_INCLUDEDIR}/opus
)

write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    "cmake/template/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
