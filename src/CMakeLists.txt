add_compile_options (--std=gnu11 -O3 -Wall -Wextra)

add_definitions (-D_GNU_SOURCE)
set (THREAD_LIBRARY pthread)
if (CMAKE_SYSTEM_NAME MATCHES Android)
    set (THREAD_LIBRARY "")
endif()
if (CMAKE_SYSTEM_NAME MATCHES Linux)
    set (EXPORTS_LINKER_FLAGS "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/libnrsc5.map")
elseif (CMAKE_SYSTEM_NAME MATCHES Darwin)
    set (CMAKE_MACOSX_RPATH ON)
    set (EXPORTS_LINKER_FLAGS "-exported_symbols_list ${CMAKE_CURRENT_SOURCE_DIR}/libnrsc5.sym")
elseif (CMAKE_SYSTEM_NAME MATCHES Windows)
    set (EXPORTS_LINKER_FLAGS "-Wl,--exclude-all-symbols")
    set (SOCKET_LIBRARY ws2_32)
endif()
if (USE_STATIC)
    set (STATIC_LINKER_FLAGS -static)
endif()

configure_file (config.h.in config.h)
include_directories (
    ${FAAD2_INCLUDE_DIRS}
    ${AO_INCLUDE_DIRS}
    ${RTL_SDR_INCLUDE_DIRS}
    ${FFTW3F_INCLUDE_DIRS}
    "${CMAKE_CURRENT_BINARY_DIR}"
    "${CMAKE_SOURCE_DIR}/include"
)
link_directories (
    ${FAAD2_LIBRARY_DIRS}
    ${AO_LIBRARY_DIRS}
    ${RTL_SDR_LIBRARY_DIRS}
    ${FFTW3F_LIBRARY_DIRS}
    ${LIBUSB_LIBRARY_DIRS}
)

set (LIBRARY_FILES
    acquire.c
    decode.c
    frame.c
    here_images.c
    input.c
    nrsc5.c
    output.c
    pids.c
    rtltcp.c
    sync.c

    firdecim_q15.c

    conv_dec.c

    rs_init.c
    rs_decode.c

    unicode.c

    strndup.c
)

set (
    LibraryDependencies
    ${FAAD2_LIBRARIES}
    ${FFTW3F_LIBRARIES}
    ${RTL_SDR_LIBRARIES}
    ${THREAD_LIBRARY}
    ${SOCKET_LIBRARY}
    m
)

add_library (
    nrsc5 SHARED
    ${LIBRARY_FILES}
)
target_link_libraries (
    nrsc5
    ${LibraryDependencies}
)
if (BUILTIN_LIBRARIES)
    add_dependencies(nrsc5 ${BUILTIN_LIBRARIES})
endif ()
set_target_properties(nrsc5 PROPERTIES PUBLIC_HEADER "../include/nrsc5.h")
set_target_properties(nrsc5 PROPERTIES LINK_FLAGS "${STATIC_LINKER_FLAGS} ${EXPORTS_LINKER_FLAGS}")
target_compile_definitions(nrsc5 PUBLIC "-DNRSC5_EXPORTS=1")
if (NOT ENABLE_RTLSDR_DEVICE)
    target_compile_definitions(nrsc5 PRIVATE NRSC5_NO_RTLSDR=1)
endif()

add_library (
    nrsc5_static
    ${LIBRARY_FILES}
)
target_link_libraries (
    nrsc5_static
    ${LibraryDependencies}
)
if (BUILTIN_LIBRARIES)
    add_dependencies(nrsc5_static ${BUILTIN_LIBRARIES})
endif ()
if (NOT ENABLE_RTLSDR_DEVICE)
    target_compile_definitions(nrsc5_static PRIVATE NRSC5_NO_RTLSDR=1)
endif()

if (BUILD_CLI)
    add_executable (
        app
        main.c
        log.c
    )
    set_property (TARGET app PROPERTY OUTPUT_NAME nrsc5)
    set_target_properties(app PROPERTIES LINK_FLAGS "${STATIC_LINKER_FLAGS}")
    target_link_libraries (
        app
        nrsc5_static
        ${AO_LIBRARIES}
        ${THREAD_LIBRARY}
        ${SOCKET_LIBRARY}
    )
    if (NOT CLI_AUDIO_OUTPUT)
        target_compile_definitions(app PRIVATE NRSC5_NO_AO=1)
    endif ()
    if (NOT ENABLE_RTLSDR_DEVICE)
        target_compile_definitions(app PRIVATE NRSC5_NO_RTLSDR=1)
    endif ()
    install (
            TARGETS app
            RUNTIME DESTINATION bin
    )
endif ()

install (
    TARGETS nrsc5 nrsc5_static
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    PUBLIC_HEADER DESTINATION include
)
