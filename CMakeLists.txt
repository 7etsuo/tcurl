cmake_minimum_required(VERSION 3.10)
project(socket VERSION 1.0.0 LANGUAGES C)

# Generate compile_commands.json for IDE integration (clangd)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Set C standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)  # Enable GNU extensions (_GNU_SOURCE)

# Compiler flags
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror -Wno-unused-function -Wno-unused-variable -Wno-comment -Wno-implicit-int -Wno-ignored-qualifiers -Wno-return-type -Wno-error=return-type -D_GNU_SOURCE -pthread -fno-strict-aliasing -fPIC")

# Executable stack warning is expected and harmless:
# - Caused by setjmp/longjmp in TRY/EXCEPT/FINALLY exception handling
# - Modern Linux kernels enforce NX protection regardless of this flag
# - This is a legacy warning from early 2000s security implementations
# - The code is completely safe despite the warning
set(CMAKE_C_FLAGS_DEBUG "-g -Og")
set(CMAKE_C_FLAGS_RELEASE "-O3 -DNDEBUG")

# Sanitizer options for debugging memory/undefined behavior issues
option(ENABLE_ASAN "Enable AddressSanitizer (ASan)" OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer (UBSan)" OFF)
option(ENABLE_SANITIZERS "Enable both ASan and UBSan" OFF)

# If ENABLE_SANITIZERS is ON, enable both
if(ENABLE_SANITIZERS)
    set(ENABLE_ASAN ON)
    set(ENABLE_UBSAN ON)
endif()

# AddressSanitizer flags
if(ENABLE_ASAN)
    message(STATUS "AddressSanitizer (ASan) enabled")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address -fno-omit-frame-pointer")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=address")
endif()

# UndefinedBehaviorSanitizer flags
if(ENABLE_UBSAN)
    message(STATUS "UndefinedBehaviorSanitizer (UBSan) enabled")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=undefined -fno-omit-frame-pointer")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=undefined")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=undefined")
endif()

# ThreadSanitizer option (incompatible with ASan)
option(ENABLE_TSAN "Enable ThreadSanitizer (TSan) - incompatible with ASan" OFF)

if(ENABLE_TSAN)
    if(ENABLE_ASAN)
        message(FATAL_ERROR "ThreadSanitizer (TSan) is incompatible with AddressSanitizer (ASan). "
                            "Use only one at a time.")
    endif()
    message(STATUS "ThreadSanitizer (TSan) enabled")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=thread -fno-omit-frame-pointer -g")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=thread")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fsanitize=thread")
endif()

# Code coverage option
option(ENABLE_COVERAGE "Enable code coverage with gcov" OFF)

if(ENABLE_COVERAGE)
    message(STATUS "Code coverage enabled (gcov)")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage -DENABLE_GCOV_FLUSH")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} --coverage")
endif()

# Default to Debug build
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif()

# Include directories
include_directories(${CMAKE_SOURCE_DIR}/include)

# Detect platform and select appropriate poll backend
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(POLL_BACKEND "epoll")
    set(POLL_BACKEND_SRC "src/poll/SocketPoll_epoll.c")
    message(STATUS "Building with epoll backend for Linux")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(POLL_BACKEND "kqueue")
    set(POLL_BACKEND_SRC "src/poll/SocketPoll_kqueue.c")
    message(STATUS "Building with kqueue backend for macOS")
elseif(CMAKE_SYSTEM_NAME MATCHES "BSD")
    set(POLL_BACKEND "kqueue")
    set(POLL_BACKEND_SRC "src/poll/SocketPoll_kqueue.c")
    message(STATUS "Building with kqueue backend for BSD")
else()
    set(POLL_BACKEND "poll")
    set(POLL_BACKEND_SRC "src/poll/SocketPoll_poll.c")
    message(STATUS "Building with poll(2) fallback backend")
endif()

# Detect OpenSSL/LibreSSL for TLS support (optional)
set(SOCKET_HAS_TLS OFF)
set(OPENSSL_LIBRARIES "")
option(ENABLE_TLS "Enable TLS/SSL support" ON)

if(ENABLE_TLS)
    find_package(OpenSSL QUIET)
    if(OpenSSL_FOUND)
        set(SOCKET_HAS_TLS ON)
        set(OPENSSL_LIBRARIES OpenSSL::SSL OpenSSL::Crypto)
        add_definitions(-DSOCKET_HAS_TLS)
        message(STATUS "OpenSSL found - TLS support enabled")
    else()
        # Try LibreSSL as fallback
        find_library(CRYPTO_LIB crypto)
        find_library(SSL_LIB ssl)
        if(CRYPTO_LIB AND SSL_LIB)
            set(SOCKET_HAS_TLS ON)
            set(OPENSSL_LIBRARIES ${SSL_LIB} ${CRYPTO_LIB})
            add_definitions(-DSOCKET_HAS_TLS)
            message(STATUS "LibreSSL found - TLS support enabled")
        else()
            message(WARNING "TLS requested but no SSL library found (OpenSSL/LibreSSL)")
        endif()
    endif()
else()
    add_definitions(-DSOCKET_HAS_TLS=0)
    message(STATUS "TLS support disabled (use -DENABLE_TLS=ON to enable)")
endif()

# io_uring support (Linux only - requires liburing and kernel 5.1+)
set(SOCKET_HAS_IO_URING OFF)
set(LIBURING_LIBRARIES "")
option(ENABLE_IO_URING "Enable io_uring async I/O backend (Linux only)" OFF)
option(PREFER_IO_URING_POLL "Use io_uring instead of epoll for poll backend (Linux only)" OFF)

if(ENABLE_IO_URING)
    if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux")
        message(WARNING "io_uring is only available on Linux - ignoring ENABLE_IO_URING")
    else()
        # Try pkg-config first (preferred)
        find_package(PkgConfig QUIET)
        if(PKG_CONFIG_FOUND)
            pkg_check_modules(LIBURING QUIET liburing)
        endif()

        if(LIBURING_FOUND)
            set(SOCKET_HAS_IO_URING ON)
            add_definitions(-DSOCKET_HAS_IO_URING=1)
            include_directories(${LIBURING_INCLUDE_DIRS})
            set(LIBURING_LIBRARIES ${LIBURING_LIBRARIES})
            message(STATUS "io_uring found via pkg-config - async I/O enabled")
        else()
            # Fallback: direct library/header search
            find_library(URING_LIB uring)
            find_path(URING_INCLUDE_DIR liburing.h)

            if(URING_LIB AND URING_INCLUDE_DIR)
                set(SOCKET_HAS_IO_URING ON)
                add_definitions(-DSOCKET_HAS_IO_URING=1)
                include_directories(${URING_INCLUDE_DIR})
                set(LIBURING_LIBRARIES ${URING_LIB})
                message(STATUS "io_uring found - async I/O enabled (${URING_LIB})")
            else()
                message(WARNING "io_uring requested but liburing not found")
                message(STATUS "  Install with: apt install liburing-dev")
            endif()
        endif()
    endif()
else()
    message(STATUS "io_uring disabled (use -DENABLE_IO_URING=ON on Linux to enable)")
endif()

# Update poll backend to io_uring if preferred and available
if(PREFER_IO_URING_POLL)
    if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux")
        message(WARNING "io_uring poll backend only available on Linux - using default backend")
    elseif(NOT SOCKET_HAS_IO_URING)
        message(WARNING "io_uring poll backend requested but liburing not found - using epoll")
        message(STATUS "  Enable with: cmake -DENABLE_IO_URING=ON -DPREFER_IO_URING_POLL=ON")
    else()
        set(POLL_BACKEND "io_uring")
        set(POLL_BACKEND_SRC "src/poll/SocketPoll_uring.c")
        message(STATUS "Switching to io_uring poll backend (preferred over epoll)")
    endif()
endif()

# HTTP Compression support (optional - requires zlib and/or brotli)
option(ENABLE_HTTP_COMPRESSION "Enable HTTP gzip/deflate/brotli compression" OFF)

set(SOCKETHTTP1_HAS_COMPRESSION OFF)
set(HTTP_COMPRESSION_LIBS "")

# Always search for zlib (needed for WebSocket permessage-deflate even without HTTP compression)
find_package(ZLIB QUIET)

# WebSocket deflate requires zlib - we'll set this after HTTP compression logic
# to avoid linking zlib twice when both WS deflate and HTTP compression use it
set(WS_DEFLATE_LIBS "")

if(ENABLE_HTTP_COMPRESSION)
    # Use zlib for HTTP compression if found
    if(ZLIB_FOUND)
        add_definitions(-DSOCKETHTTP1_HAS_ZLIB)
        list(APPEND HTTP_COMPRESSION_LIBS ZLIB::ZLIB)
        message(STATUS "zlib found - gzip/deflate compression enabled")
        # zlib already in HTTP_COMPRESSION_LIBS, no need to duplicate in WS_DEFLATE_LIBS
    else()
        message(STATUS "zlib not found - gzip/deflate compression disabled")
    endif()
    
    # Find Brotli (optional)
    find_library(BROTLI_DEC_LIB brotlidec)
    find_library(BROTLI_ENC_LIB brotlienc)
    find_path(BROTLI_INCLUDE_DIR brotli/decode.h)
    
    if(BROTLI_DEC_LIB AND BROTLI_ENC_LIB AND BROTLI_INCLUDE_DIR)
        add_definitions(-DSOCKETHTTP1_HAS_BROTLI)
        list(APPEND HTTP_COMPRESSION_LIBS ${BROTLI_DEC_LIB} ${BROTLI_ENC_LIB})
        include_directories(${BROTLI_INCLUDE_DIR})
        message(STATUS "Brotli found - brotli compression enabled")
    else()
        message(STATUS "Brotli not found - brotli compression disabled")
    endif()
    
    # Enable compression module if any compression library found
    if(ZLIB_FOUND OR (BROTLI_DEC_LIB AND BROTLI_ENC_LIB))
        set(SOCKETHTTP1_HAS_COMPRESSION ON)
        message(STATUS "HTTP compression support enabled")
    else()
        message(WARNING "HTTP compression requested but no compression libraries found")
    endif()
else()
    message(STATUS "HTTP compression disabled (use -DENABLE_HTTP_COMPRESSION=ON to enable)")
    # HTTP compression disabled, so WS deflate needs its own zlib link
    if(ZLIB_FOUND)
        set(WS_DEFLATE_LIBS ZLIB::ZLIB)
    endif()
endif()

if(SOCKETHTTP1_HAS_COMPRESSION)
    add_definitions(-DSOCKETHTTP1_HAS_COMPRESSION=1)
else()
    add_definitions(-DSOCKETHTTP1_HAS_COMPRESSION=0)
endif()

# Collect all source files
set(LIB_SOURCES
    # Core foundation (consolidated)
    src/core/Arena.c
    src/core/Except.c
    src/core/SocketUtil.c
    src/core/SocketTimer.c
    src/core/SocketRateLimit.c
    src/core/HashTable.c
    src/core/TimeWindow.c
    src/core/SocketIPTracker.c
    src/core/SocketSYNProtect.c
    src/core/SocketSYNProtect-list.c
    src/core/SocketSYNProtect-ip.c
    src/core/SocketCrypto.c
    src/core/SocketUTF8.c
    src/core/SocketSecurity.c
    src/core/SocketMetrics.c
    src/core/SocketRetry.c
    
    # Socket core (consolidated)
    src/socket/Socket.c
    src/socket/Socket-connect.c
    src/socket/Socket-convenience.c
    src/socket/Socket-fd.c
    src/socket/Socket-iov.c
    src/socket/Socket-options.c
    src/socket/SocketDgram.c
    src/socket/SocketBuf.c
    src/socket/SocketCommon.c
    src/socket/SocketAsync.c
    src/socket/SocketIO.c
    src/socket/SocketHappyEyeballs.c
    src/socket/SocketReconnect.c
    
    # Proxy module (Phase 8)
    src/socket/SocketProxy.c
    src/socket/SocketProxy-http.c
    src/socket/SocketProxy-socks4.c
    src/socket/SocketProxy-socks5.c
    
    # WebSocket module (Phase 9 - RFC 6455)
    src/socket/SocketWS.c
    src/socket/SocketWS-handshake.c
    src/socket/SocketWS-frame.c
    src/socket/SocketWS-transport.c

    # WebSocket over HTTP/2 (RFC 8441)
    src/socket/SocketWSH2.c
    
    # DNS module (consolidated)
    src/dns/SocketDNS.c
    src/dns/SocketDNS-internal.c
    src/dns/SocketDNSWire.c
    src/dns/SocketDNSTransport.c
    src/dns/SocketDNSConfig.c
    src/dns/SocketDNSResolver.c
    src/dns/SocketDNSSEC.c
    src/dns/SocketDNSCookie.c
    src/dns/SocketDNSError.c
    src/dns/SocketDNSNegCache.c
    src/dns/SocketDNSServfailCache.c
    src/dns/SocketDNSDeadServer.c

    # Poll module (consolidated)
    src/poll/SocketPoll.c
    ${POLL_BACKEND_SRC}
    
    # Pool module (consolidated)
    src/pool/SocketPool-core.c
    src/pool/SocketPool-connections.c
    src/pool/SocketPool-ops.c
    src/pool/SocketPool-ratelimit.c
    src/pool/SocketPool-drain.c
    src/pool/SocketPool-health.c
    
    # HTTP module (RFC 9110/3986)
    src/http/SocketHTTP-core.c
    src/http/SocketHTTP-headers.c
    src/http/SocketHTTP-uri.c
    src/http/SocketHTTP-date.c
    
    # HTTP/1.1 module (RFC 9112)
    src/http/SocketHTTP1-parser.c
    src/http/SocketHTTP1-serialize.c
    src/http/SocketHTTP1-chunked.c
    
    # HPACK Header Compression (RFC 7541)
    src/http/SocketHPACK.c
    src/http/SocketHPACK-huffman.c
    src/http/SocketHPACK-table.c
    
    # HTTP/2 Protocol (RFC 9113)
    src/http/SocketHTTP2-frame.c
    src/http/SocketHTTP2-connection.c
    src/http/SocketHTTP2-stream.c
    src/http/SocketHTTP2-flow.c
    src/http/SocketHTTP2-validate.c
    src/http/SocketHTTP2-priority.c

    # HTTP Client and Server APIs (Phase 7)
    src/http/SocketHTTPClient.c
    src/http/SocketHTTPClient-pool.c
    src/http/SocketHTTPClient-auth.c
    src/http/SocketHTTPClient-cookie.c
    src/http/SocketHTTPClient-retry.c
    src/http/SocketHTTPClient-arena.c
    src/http/SocketHTTPClient-async.c
    src/http/SocketHTTPServer.c
    src/http/SocketHTTPServer-connections.c

    # Streaming Curl module
    # Phase 1: URL parsing and core types
    src/curl/StreamingCurl_url.c
    # Phase 2: Connection establishment (Happy Eyeballs, Proxy, TLS/ALPN)
    src/curl/StreamingCurl_connect.c
    # Phase 3: Request building and sending (HTTP/1.1 and HTTP/2)
    src/curl/StreamingCurl_request.c
    src/curl/StreamingCurl_auth.c
    # Phase 4: Response streaming (HTTP/1.1 and HTTP/2)
    src/curl/StreamingCurl_h1.c
    src/curl/StreamingCurl_h2.c
    src/curl/StreamingCurl_decomp.c
    # Phase 5: Redirect and cookie handling
    src/curl/StreamingCurl_redirect.c
    src/curl/StreamingCurl_cookie.c
    # Phase 6: Public API and session management
    src/curl/StreamingCurl.c
    # Phase 7: CLI argument parsing (library component)
    src/curl/curl_args.c
)

# Add TLS sources if enabled (split for maintainability)
if(SOCKET_HAS_TLS)
    list(APPEND LIB_SOURCES
        src/tls/SocketTLS.c
        src/tls/SocketTLS-ktls.c
        src/tls/SocketTLS-performance.c
        src/tls/SocketTLSContext-core.c
        src/tls/SocketTLSContext-certs.c
        src/tls/SocketTLSContext-alpn.c
        src/tls/SocketTLSContext-session.c
        src/tls/SocketTLSContext-verify.c
        src/tls/SocketTLSContext-pinning.c
        src/tls/SocketTLSConfig.c
        src/tls/SocketTLSContext-ct.c
        src/tls/SocketTLSContext-crl.c
        # DTLS (Datagram TLS) support
        src/tls/SocketDTLS.c
        src/tls/SocketDTLSContext.c
        src/tls/SocketDTLS-cookie.c
        # DNS-over-TLS (RFC 7858)
        src/dns/SocketDNSoverTLS.c
        # DNS-over-HTTPS (RFC 8484)
        src/dns/SocketDNSoverHTTPS.c
    )
endif()

# Add HTTP compression source if compression enabled
if(ENABLE_HTTP_COMPRESSION AND (ZLIB_FOUND OR (BROTLI_DEC_LIB AND BROTLI_ENC_LIB)))
    list(APPEND LIB_SOURCES
        src/http/SocketHTTP1-compress.c
    )
endif()

# Add WebSocket compression source if zlib available
if(ZLIB_FOUND)
    list(APPEND LIB_SOURCES
        src/socket/SocketWS-deflate.c
    )
    add_definitions(-DSOCKETWS_HAS_DEFLATE)
    message(STATUS "WebSocket permessage-deflate compression enabled")
endif()

# Object library for reuse in tests (avoids static library constructor issues)
add_library(socket_objects OBJECT ${LIB_SOURCES})
target_include_directories(socket_objects PRIVATE ${CMAKE_SOURCE_DIR}/include)

# Add OpenSSL include directories to object library when TLS is enabled
if(SOCKET_HAS_TLS AND OpenSSL_FOUND)
    target_include_directories(socket_objects PRIVATE ${OPENSSL_INCLUDE_DIR})
endif()

# Static library
add_library(socket_static STATIC $<TARGET_OBJECTS:socket_objects>)
set_target_properties(socket_static PROPERTIES OUTPUT_NAME socket)
if(SOCKET_HAS_TLS)
    target_link_libraries(socket_static pthread m dl ${LIBURING_LIBRARIES} ${OPENSSL_LIBRARIES} ${HTTP_COMPRESSION_LIBS} ${WS_DEFLATE_LIBS})
else()
    target_link_libraries(socket_static pthread m ${LIBURING_LIBRARIES} ${HTTP_COMPRESSION_LIBS} ${WS_DEFLATE_LIBS})
endif()

# Shared library
add_library(socket_shared SHARED $<TARGET_OBJECTS:socket_objects>)
set_target_properties(socket_shared PROPERTIES OUTPUT_NAME socket)
if(SOCKET_HAS_TLS)
    target_link_libraries(socket_shared pthread m dl ${LIBURING_LIBRARIES} ${OPENSSL_LIBRARIES} ${HTTP_COMPRESSION_LIBS} ${WS_DEFLATE_LIBS})
else()
    target_link_libraries(socket_shared pthread m ${LIBURING_LIBRARIES} ${HTTP_COMPRESSION_LIBS} ${WS_DEFLATE_LIBS})
endif()

# Custom target to build both libraries (default)
add_custom_target(lib ALL DEPENDS socket_static socket_shared)

# Generate pkg-config file
set(PC_LIBS_PRIVATE "")
if(SOCKET_HAS_TLS)
    set(PC_LIBS_PRIVATE "-lssl -lcrypto")
endif()

configure_file(
    ${CMAKE_SOURCE_DIR}/libsocket.pc.in
    ${CMAKE_BINARY_DIR}/libsocket.pc
    @ONLY
)

install(FILES ${CMAKE_BINARY_DIR}/libsocket.pc
    DESTINATION lib/pkgconfig
)

# Installation targets (optional)

# Header files for installation
set(CORE_HEADERS
    include/core/Arena.h
    include/core/Except.h
    include/core/SocketConfig.h
    include/core/SocketUtil.h
    include/core/SocketTimer.h
    include/core/SocketTimer-private.h
    include/core/SocketRateLimit.h
    include/core/SocketRateLimit-private.h
    include/core/SocketIPTracker.h
    include/core/SocketSYNProtect.h
    include/core/SocketSYNProtect-private.h
    include/core/SocketCrypto.h
    include/core/SocketUTF8.h
    include/core/SocketSecurity.h
)

set(SOCKET_HEADERS
    include/socket/Socket.h
    include/socket/Socket-private.h
    include/socket/SocketAsync.h
    include/socket/SocketBuf.h
    include/socket/SocketCommon.h
    include/socket/SocketDgram.h
    include/socket/SocketIO.h
    include/socket/SocketHappyEyeballs.h
    include/socket/SocketHappyEyeballs-private.h
    include/socket/SocketReconnect.h
    include/socket/SocketReconnect-private.h
    include/socket/SocketProxy.h
    include/socket/SocketProxy-private.h
)

set(DNS_HEADERS
    include/dns/SocketDNS.h
    include/dns/SocketDNSWire.h
    include/dns/SocketDNSTransport.h
    include/dns/SocketDNSConfig.h
    include/dns/SocketDNSResolver.h
    include/dns/SocketDNSoverTLS.h
    include/dns/SocketDNSoverHTTPS.h
    include/dns/SocketDNSSEC.h
    include/dns/SocketDNSCookie.h
    include/dns/SocketDNSError.h
    include/dns/SocketDNSNegCache.h
    include/dns/SocketDNSServfailCache.h
    include/dns/SocketDNSDeadServer.h
)

set(POLL_HEADERS
    include/poll/SocketPoll.h
    include/poll/SocketPoll-private.h
    include/poll/SocketPoll_backend.h
)

set(POOL_HEADERS
    include/pool/SocketPool.h
    include/pool/SocketPool-private.h
    include/pool/SocketPool-core.h
)

set(TLS_HEADERS
    include/tls/SocketTLS.h
    include/tls/SocketTLSConfig.h
    include/tls/SocketTLSContext.h
    include/tls/SocketDTLS.h
    include/tls/SocketDTLSConfig.h
    include/tls/SocketDTLSContext.h
    include/tls/SocketDTLS-private.h
)

set(HTTP_HEADERS
    include/http/SocketHTTP.h
    include/http/SocketHTTP-private.h
    include/http/SocketHTTP1.h
    include/http/SocketHTTP1-private.h
    include/http/SocketHPACK.h
    include/http/SocketHPACK-private.h
    include/http/SocketHTTP2.h
    include/http/SocketHTTP2-private.h
    include/http/SocketHTTPClient.h
    include/http/SocketHTTPClient-private.h
    include/http/SocketHTTPServer.h
)

set(CURL_HEADERS
    include/curl/StreamingCurl.h
    include/curl/StreamingCurl-private.h
    include/curl/curl_args.h
)

install(TARGETS socket_static socket_shared
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
)

install(FILES ${CORE_HEADERS} DESTINATION include/core)
install(FILES ${SOCKET_HEADERS} DESTINATION include/socket)
install(FILES ${DNS_HEADERS} DESTINATION include/dns)
install(FILES ${POLL_HEADERS} DESTINATION include/poll)
install(FILES ${POOL_HEADERS} DESTINATION include/pool)
install(FILES ${TLS_HEADERS} DESTINATION include/tls)
install(FILES ${HTTP_HEADERS} DESTINATION include/http)
install(FILES ${CURL_HEADERS} DESTINATION include/curl)

# Print build configuration
message(STATUS "Socket Library Configuration:")
message(STATUS "  Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "  C Compiler: ${CMAKE_C_COMPILER}")
message(STATUS "  C Flags: ${CMAKE_C_FLAGS}")
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    message(STATUS "  Debug Flags: ${CMAKE_C_FLAGS_DEBUG}")
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    message(STATUS "  Release Flags: ${CMAKE_C_FLAGS_RELEASE}")
endif()
message(STATUS "  Poll Backend: ${POLL_BACKEND}")
if(SOCKET_HAS_TLS)
    message(STATUS "  TLS Support: Enabled (${OPENSSL_LIBRARIES})")
else()
    message(STATUS "  TLS Support: Disabled")
endif()
if(SOCKET_HAS_IO_URING)
    message(STATUS "  io_uring: Enabled (${LIBURING_LIBRARIES})")
else()
    message(STATUS "  io_uring: Disabled")
endif()
if(ENABLE_ASAN)
    message(STATUS "  AddressSanitizer: Enabled")
endif()
if(ENABLE_UBSAN)
    message(STATUS "  UBSanitizer: Enabled")
endif()
if(ENABLE_COVERAGE)
    message(STATUS "  Code Coverage: Enabled")
endif()

# Documentation generation (Doxygen)
find_package(Doxygen)
if(DOXYGEN_FOUND)
    set(DOXYGEN_IN ${CMAKE_SOURCE_DIR}/Doxyfile)
    set(DOXYGEN_OUT ${CMAKE_BINARY_DIR}/docs)
    
    # Copy CSS file to HTML output directory (Doxygen will include it)
    add_custom_command(
        OUTPUT ${CMAKE_SOURCE_DIR}/docs/html/doxygen-awesome.css
        COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_SOURCE_DIR}/docs/html
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_SOURCE_DIR}/docs/doxygen-awesome.css
            ${CMAKE_SOURCE_DIR}/docs/html/doxygen-awesome.css
        COMMENT "Preparing Doxygen CSS"
        VERBATIM
    )
    
    add_custom_target(doc
        COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_SOURCE_DIR}/docs/html
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_SOURCE_DIR}/docs/doxygen-awesome.css
            ${CMAKE_SOURCE_DIR}/docs/html/doxygen-awesome.css
        COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_IN}
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Generating API documentation with Doxygen"
        VERBATIM
    )
    
    message(STATUS "Documentation: Doxygen found - use 'make doc' to generate API docs")
else()
    message(STATUS "Documentation: Doxygen not found - install doxygen to generate API docs")
endif()

# =============================================================================
# tcurl CLI Tool
# =============================================================================
# A curl-compatible command-line HTTP client using StreamingCurl
# =============================================================================

add_executable(tcurl src/curl/curl_main.c)
target_link_libraries(tcurl socket_static pthread)
target_include_directories(tcurl PRIVATE ${CMAKE_SOURCE_DIR}/include)

install(TARGETS tcurl
    RUNTIME DESTINATION bin
)

message(STATUS "tcurl: Command-line HTTP client enabled")

# =============================================================================
# Examples (optional)
# =============================================================================
# Build with: cmake -DBUILD_EXAMPLES=ON ..
# =============================================================================

option(BUILD_EXAMPLES "Build example programs" OFF)

if(BUILD_EXAMPLES)
    add_subdirectory(examples)
    message(STATUS "Examples: Building example programs")
else()
    message(STATUS "Examples: Disabled (use -DBUILD_EXAMPLES=ON to enable)")
endif()



