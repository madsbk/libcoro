cmake_minimum_required(VERSION 3.15)

include(CMakeDependentOption)
include(GNUInstallDirs)
include(GenerateExportHeader)
include(CMakePackageConfigHelpers)

project(libcoro
  VERSION 0.15.0
  DESCRIPTION "C++20 coroutine library"
  LANGUAGES CXX
)

# ------------------------------------------------------------------------------
# Options
# ------------------------------------------------------------------------------
option(LIBCORO_ENABLE_ASAN            "Build with AddressSanitizer" OFF)
option(LIBCORO_ENABLE_MSAN            "Build with MemorySanitizer" OFF)
option(LIBCORO_ENABLE_TSAN            "Build with ThreadSanitizer" OFF)
option(LIBCORO_ENABLE_USAN            "Build with UndefinedBehaviorSanitizer" OFF)

option(LIBCORO_EXTERNAL_DEPENDENCIES  "Use find_package() for deps instead of vendored ones" OFF)
option(LIBCORO_BUILD_TESTS            "Build tests" ON)
option(LIBCORO_BUILD_EXAMPLES         "Build examples" ON)
option(LIBCORO_CODE_COVERAGE          "Enable coverage (tests must be on)" OFF)

cmake_dependent_option(LIBCORO_FEATURE_NETWORKING "Include networking features" ON "NOT EMSCRIPTEN; NOT MSVC" OFF)
cmake_dependent_option(LIBCORO_FEATURE_TLS        "Include TLS features"       ON "LIBCORO_FEATURE_NETWORKING; NOT EMSCRIPTEN; NOT MSVC" OFF)

if(UNIX AND NOT APPLE)
  set(LINUX TRUE)
endif()
if(APPLE)
  set(MACOSX TRUE)
endif()

# ------------------------------------------------------------------------------
# Dependencies (build-time)
# ------------------------------------------------------------------------------
if(LIBCORO_EXTERNAL_DEPENDENCIES)
  if(LIBCORO_FEATURE_NETWORKING)
    find_package(c-ares CONFIG REQUIRED) # c-ares::cares
  endif()
  if(LIBCORO_FEATURE_TLS)
    find_package(OpenSSL REQUIRED)       # OpenSSL::SSL OpenSSL::Crypto
  endif()
else()
  if(LIBCORO_FEATURE_NETWORKING)
    # Prevent vendored c-ares from installing
    set(CARES_INSTALL OFF CACHE INTERNAL "")
    # Mirror BUILD_SHARED_LIBS for c-ares without touching parent cache
    if(BUILD_SHARED_LIBS)
      set(CARES_SHARED ON  CACHE INTERNAL "")
      set(CARES_STATIC OFF CACHE INTERNAL "")
    else()
      set(CARES_SHARED OFF CACHE INTERNAL "")
      set(CARES_STATIC ON  CACHE INTERNAL "")
    endif()
    add_subdirectory(vendor/c-ares/c-ares)
  endif()
endif()

# ------------------------------------------------------------------------------
# Sources
# ------------------------------------------------------------------------------
set(LIBCORO_SOURCE_FILES
  include/coro/concepts/awaitable.hpp
  include/coro/concepts/buffer.hpp
  include/coro/concepts/executor.hpp
  include/coro/concepts/promise.hpp
  include/coro/concepts/range_of.hpp

  include/coro/detail/awaiter_list.hpp
  include/coro/detail/task_self_deleting.hpp  src/detail/task_self_deleting.cpp
  include/coro/detail/void_value.hpp

  include/coro/attribute.hpp
  include/coro/condition_variable.hpp         src/condition_variable.cpp
  include/coro/coro.hpp
  include/coro/default_executor.hpp            src/default_executor.cpp
  include/coro/event.hpp                       src/event.cpp
  include/coro/generator.hpp
  include/coro/latch.hpp
  include/coro/mutex.hpp                       src/mutex.cpp
  include/coro/queue.hpp
  include/coro/ring_buffer.hpp
  include/coro/semaphore.hpp                   src/semaphore.cpp
  include/coro/shared_mutex.hpp
  include/coro/sync_wait.hpp                   src/sync_wait.cpp
  include/coro/task_container.hpp
  include/coro/task.hpp
  include/coro/thread_pool.hpp                 src/thread_pool.cpp
  include/coro/time.hpp
  include/coro/when_all.hpp
  include/coro/when_any.hpp
)

if(LIBCORO_FEATURE_NETWORKING)
  list(APPEND LIBCORO_SOURCE_FILES
    include/coro/detail/poll_info.hpp
    include/coro/detail/timer_handle.hpp       src/detail/timer_handle.cpp

    include/coro/fd.hpp
    include/coro/io_scheduler.hpp              src/io_scheduler.cpp
    include/coro/io_notifier.hpp
    include/coro/poll.hpp                      src/poll.cpp
  )
  if(LINUX)
    list(APPEND LIBCORO_SOURCE_FILES
      include/coro/detail/io_notifier_epoll.hpp  src/detail/io_notifier_epoll.cpp
    )
  endif()
  if(MACOSX)
    list(APPEND LIBCORO_SOURCE_FILES
      include/coro/detail/io_notifier_kqueue.hpp src/detail/io_notifier_kqueue.cpp
    )
  endif()

  list(APPEND LIBCORO_SOURCE_FILES
    include/coro/net/dns/resolver.hpp          src/net/dns/resolver.cpp
    include/coro/net/connect.hpp               src/net/connect.cpp
    include/coro/net/hostname.hpp
    include/coro/net/ip_address.hpp            src/net/ip_address.cpp
    include/coro/net/recv_status.hpp           src/net/recv_status.cpp
    include/coro/net/send_status.hpp           src/net/send_status.cpp
    include/coro/net/socket.hpp                src/net/socket.cpp
    include/coro/net/tcp/client.hpp            src/net/tcp/client.cpp
    include/coro/net/tcp/server.hpp            src/net/tcp/server.cpp
    include/coro/net/udp/peer.hpp              src/net/udp/peer.cpp
  )

  if(LIBCORO_FEATURE_TLS)
    list(APPEND LIBCORO_SOURCE_FILES
      include/coro/net/tls/client.hpp            src/net/tls/client.cpp
      include/coro/net/tls/connection_status.hpp src/net/tls/connection_status.cpp
      include/coro/net/tls/context.hpp           src/net/tls/context.cpp
      include/coro/net/tls/recv_status.hpp       src/net/tls/recv_status.cpp
      include/coro/net/tls/send_status.hpp       src/net/tls/send_status.cpp
      include/coro/net/tls/server.hpp            src/net/tls/server.cpp
    )
  endif()
endif()

if(DEFINED EMSCRIPTEN)
  add_compile_options(-fwasm-exceptions -pthread -matomics)
  add_link_options(-fwasm-exceptions -pthread -sPROXY_TO_PTHREAD -sALLOW_MEMORY_GROWTH -sEXIT_RUNTIME)
endif()

# ------------------------------------------------------------------------------
# Library (let BUILD_SHARED_LIBS decide; no explicit SHARED/STATIC)
# ------------------------------------------------------------------------------
add_library(libcoro ${LIBCORO_SOURCE_FILES})

# Modern C++
target_compile_features(libcoro PUBLIC cxx_std_20)

# Public includes (build & install)
target_include_directories(libcoro
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Export header
generate_export_header(libcoro BASE_NAME CORO EXPORT_FILE_NAME include/coro/export.hpp)

# Platform libs
if(UNIX)
  target_link_libraries(libcoro PUBLIC pthread)
endif()

# Feature deps
if(LIBCORO_FEATURE_NETWORKING)
  if(LIBCORO_EXTERNAL_DEPENDENCIES)
    target_link_libraries(libcoro PUBLIC c-ares::cares)
  else()
    target_link_libraries(libcoro PUBLIC c-ares) # vendored target
  endif()
  target_compile_definitions(libcoro PUBLIC LIBCORO_FEATURE_NETWORKING)
  if(LIBCORO_FEATURE_TLS)
    target_link_libraries(libcoro PUBLIC OpenSSL::SSL OpenSSL::Crypto)
    target_compile_definitions(libcoro PUBLIC LIBCORO_FEATURE_TLS)
  endif()
endif()

# Sanitizers (opt-in)
if(LIBCORO_ENABLE_ASAN)
  target_compile_options(libcoro PRIVATE -g -O0 -fno-omit-frame-pointer -fsanitize=address)
  target_link_options(libcoro    PRIVATE -fsanitize=address)
endif()
if(LIBCORO_ENABLE_MSAN)
  target_compile_options(libcoro PRIVATE -g -O0 -fno-omit-frame-pointer -fsanitize=memory)
  target_link_options(libcoro    PRIVATE -fsanitize=memory)
endif()
if(LIBCORO_ENABLE_TSAN)
  target_compile_options(libcoro PRIVATE -g -O0 -fno-omit-frame-pointer -fsanitize=thread)
  target_link_options(libcoro    PRIVATE -fsanitize=thread)
endif()
if(LIBCORO_ENABLE_USAN)
  target_compile_options(libcoro PRIVATE -g -O0 -fno-omit-frame-pointer -fsanitize=undefined)
  target_link_options(libcoro    PRIVATE -fsanitize=undefined)
endif()

# Toolchain warnings
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "10.2.0")
    message(FATAL_ERROR "g++ version ${CMAKE_CXX_COMPILER_VERSION} is unsupported, need >= 10.2.0")
  endif()
  target_compile_options(libcoro PRIVATE -fexceptions -Wall -Wextra -pipe)
elseif("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "16.0.0")
    message(FATAL_ERROR "Clang version ${CMAKE_CXX_COMPILER_VERSION} is unsupported, need >= 16.0.0")
  endif()
  target_compile_options(libcoro PRIVATE -fexceptions -Wall -Wextra -pipe)
elseif(MSVC)
  target_compile_options(libcoro PRIVATE /W4)
endif()

# Namespaced alias
add_library(libcoro::libcoro ALIAS libcoro)

# ------------------------------------------------------------------------------
# Tests / Examples
# ------------------------------------------------------------------------------
if(LIBCORO_BUILD_TESTS)
  if(LIBCORO_CODE_COVERAGE)
    target_compile_options(libcoro PRIVATE --coverage)
    target_link_libraries(libcoro PRIVATE gcov)
  endif()
  include(CTest)
  add_subdirectory(test)
endif()

if(LIBCORO_BUILD_EXAMPLES)
  add_subdirectory(examples)
endif()

# ------------------------------------------------------------------------------
# Installation
# ------------------------------------------------------------------------------
install(TARGETS libcoro
  EXPORT libcoro-targets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
install(DIRECTORY include/coro DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/include/coro DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# pkg-config
if(MSVC)
  set(target1 ${PROJECT_NAME})
else()
  string(REGEX REPLACE "^lib" "" target1 ${PROJECT_NAME})
endif()
configure_file(libcoro.pc.in libcoro.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libcoro.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

# CMake package config
set(LIBCORO_CONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/libcoro")

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/libcoroConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
)

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/libcoroConfig.cmake.in" [=[
@PACKAGE_INIT@
include(CMakeFindDependencyMacro)

set(LIBCORO_FEATURE_NETWORKING "@LIBCORO_FEATURE_NETWORKING@")
set(LIBCORO_FEATURE_TLS        "@LIBCORO_FEATURE_TLS@")
set(LIBCORO_EXTERNAL_DEPS      "@LIBCORO_EXTERNAL_DEPENDENCIES@")

if(LIBCORO_FEATURE_NETWORKING AND LIBCORO_EXTERNAL_DEPS)
  find_dependency(c-ares CONFIG)
endif()
if(LIBCORO_FEATURE_NETWORKING AND LIBCORO_FEATURE_TLS)
  find_dependency(OpenSSL)
endif()

include("${CMAKE_CURRENT_LIST_DIR}/libcoro-targets.cmake")
]=])

configure_package_config_file(
  "${CMAKE_CURRENT_BINARY_DIR}/libcoroConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/libcoroConfig.cmake"
  INSTALL_DESTINATION "${LIBCORO_CONFIG_INSTALL_DIR}"
)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/libcoroConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/libcoroConfigVersion.cmake"
  DESTINATION "${LIBCORO_CONFIG_INSTALL_DIR}"
)

install(
  EXPORT libcoro-targets
  NAMESPACE libcoro::
  DESTINATION "${LIBCORO_CONFIG_INSTALL_DIR}"
)

# ------------------------------------------------------------------------------
# Logging
# ------------------------------------------------------------------------------
message(STATUS "libcoro ${PROJECT_VERSION}")
message(STATUS "  BUILD_SHARED_LIBS             = ${BUILD_SHARED_LIBS}")
message(STATUS "  LIBCORO_EXTERNAL_DEPENDENCIES = ${LIBCORO_EXTERNAL_DEPENDENCIES}")
message(STATUS "  LIBCORO_FEATURE_NETWORKING    = ${LIBCORO_FEATURE_NETWORKING}")
message(STATUS "  LIBCORO_FEATURE_TLS           = ${LIBCORO_FEATURE_TLS}")
message(STATUS "  LIBCORO_BUILD_TESTS           = ${LIBCORO_BUILD_TESTS}")
message(STATUS "  LIBCORO_BUILD_EXAMPLES        = ${LIBCORO_BUILD_EXAMPLES}")
