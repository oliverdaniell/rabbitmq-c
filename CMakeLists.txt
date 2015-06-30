# vim:set ts=2 sw=2 sts=2 et:
cmake_minimum_required(VERSION 3.0.0)

include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.11.4.tar.gz"
    SHA1 "caded3d303e471f789db67e658b5a4ffb221d670"
)
project(rabbitmq-c VERSION 5.1.1 LANGUAGES "C")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# Follow all steps below in order to calculate new ABI version when updating the library
# NOTE: THIS IS UNRELATED to the actual project version
#
# 1. If the library source code has changed at all since the last update, then increment revision
# 2. If any interfaces have been added, removed, or changed since the last update, increment current and set revision to 0.
# 3. If any interfaces have been added since the last public release, then increment age.
# 4. If any interfaces have been removed since the last public release, then set age to 0.

set(RMQ_SOVERSION_CURRENT   4)
set(RMQ_SOVERSION_REVISION  1)
set(RMQ_SOVERSION_AGE       0)

math(EXPR RMQ_SOVERSION_MAJOR "${RMQ_SOVERSION_CURRENT} - ${RMQ_SOVERSION_AGE}")
math(EXPR RMQ_SOVERSION_MINOR "${RMQ_SOVERSION_AGE}")
math(EXPR RMQ_SOVERSION_PATCH "${RMQ_SOVERSION_REVISION}")

set(RMQ_VERSION ${RMQ_SOVERSION_MAJOR}.${RMQ_SOVERSION_MINOR}.${RMQ_SOVERSION_PATCH})
set(RMQ_SOVERSION ${RMQ_SOVERSION_MAJOR})

file(STRINGS librabbitmq/amqp.h _API_VERSION_MAJOR REGEX "^#define AMQP_VERSION_MAJOR [0-9]+$")
file(STRINGS librabbitmq/amqp.h _API_VERSION_MINOR REGEX "^#define AMQP_VERSION_MINOR [0-9]+$")
file(STRINGS librabbitmq/amqp.h _API_VERSION_PATCH REGEX "^#define AMQP_VERSION_PATCH [0-9]+$")

string(REGEX MATCH "[0-9]+" _API_VERSION_MAJOR ${_API_VERSION_MAJOR})
string(REGEX MATCH "[0-9]+" _API_VERSION_MINOR ${_API_VERSION_MINOR})
string(REGEX MATCH "[0-9]+" _API_VERSION_PATCH ${_API_VERSION_PATCH})

# VERSION to match what is in autotools
set(VERSION ${_API_VERSION_MAJOR}.${_API_VERSION_MINOR}.${_API_VERSION_PATCH})

include(TestCInline)
include(CheckFunctionExists)
include(CheckSymbolExists)
include(CheckLibraryExists)
include(CMakePushCheckState)
include(GNUInstallDirs)
include(CheckCCompilerFlag)

# Detect if we need to link against a socket library:
cmake_push_check_state()
if (WIN32)
  # Always use WinSock2 on Windows
  set(SOCKET_LIBRARIES ws2_32)
else ()
  # Is it in the default link?
  CHECK_FUNCTION_EXISTS(getaddrinfo HAVE_GETADDRINFO)
  if (NOT (HAVE_GETADDRINFO EQUAL 1))
    CHECK_LIBRARY_EXISTS(socket getaddrinfo "" HAVE_GETADDRINFO2)
    if (HAVE_GETADDRINFO2 EQUAL 1)
      set(SOCKET_LIBRARIES socket)
    else ()
      CHECK_LIBRARY_EXISTS("socket;nsl" getaddrinfo "" HAVE_GETADDRINFO3)
      if (HAVE_GETADDRINFO3 EQUAL 1)
        set(SOCKET_LIBRARIES socket nsl)
      else ()
        message(FATAL_ERROR "Cannot find name resolution library (containing symbol getaddrinfo)")
      endif ()
    endif ()
  endif ()

  set(CMAKE_REQUIRED_LIBRARIES ${SOCKET_LIBRARIES})
  CHECK_FUNCTION_EXISTS(socket HAVE_SOCKET)
  if (NOT HAVE_SOCKET EQUAL 1)
    CHECK_LIBRARY_EXISTS(socket socket "" HAVE_SOCKET2)
    if (NOT HAVE_SOCKET2 EQUAL 1)
      set(SOCKET_LIBRARIES socket ${SOCKET_LIBRARIES})
    else ()
      CHECK_LIBRARY_EXISTS("socket;nsl" socket "" HAVE_SOCKET3)
      if (HAVE_SOCKET3 EQUAL 1)
        set(SOCKET_LIBRARIES socket nsl ${SOCKET_LIBRARIES})
      else ()
        message(FATAL_ERROR "Cannot find socket library (containing symbol socket)")
      endif ()
    endif ()
  endif ()
endif ()
cmake_pop_check_state()

cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${SOCKET_LIBRARIES})
if (WIN32)
  check_symbol_exists(htonll Winsock2.h HAVE_HTONLL)
else (WIN32)
  check_symbol_exists(htonll arpa/inet.h HAVE_HTONLL)
endif (WIN32)
cmake_pop_check_state()

cmake_push_check_state()
set(CMAKE_REQUIRED_INCLUDES "poll.h")
check_function_exists(poll HAVE_POLL)
set(CMAKE_REQUIRED_INCLUDES )
if (NOT HAVE_POLL)
  if (WIN32)
    set(HAVE_SELECT 1)
  else()
    set(CMAKE_REQUIRED_INCLUDES sys/select.h)
    check_function_exists(select HAVE_SELECT)
    set(CMAKE_REQUIRED_INCLUDES )
  endif()
  if (NOT HAVE_SELECT)
    message(FATAL_ERROR "rabbitmq-c requires poll() or select() to be available")
  endif()
endif()
cmake_pop_check_state()

check_library_exists(rt clock_gettime "time.h" CLOCK_GETTIME_NEEDS_LIBRT)
if (CLOCK_GETTIME_NEEDS_LIBRT)
  set(LIBRT rt)
endif()

find_package(POPT)
find_package(XmlTo)
find_package(Doxygen)

if (POPT_FOUND AND XmlTo_FOUND)
  set(DO_DOCS ON)
endif()

find_package(Threads)

option(BUILD_SHARED_LIBS "Build rabbitmq-c as a shared library" OFF)
option(BUILD_STATIC_LIBS "Build rabbitmq-c as a static library" ON)

option(BUILD_EXAMPLES "Build Examples" OFF)
option(BUILD_TOOLS "Build Tools (requires POPT Library)" OFF)
option(BUILD_TOOLS_DOCS "Build man pages for Tools (requires xmlto)" OFF)
option(BUILD_TESTS "Build tests (run tests with make test)" OFF)
option(BUILD_API_DOCS "Build Doxygen API docs" ${DOXYGEN_FOUND})
option(ENABLE_SSL_SUPPORT "Enable SSL support" ON)
option(ENABLE_THREAD_SAFETY "Enable thread safety when using OpenSSL" ${Threads_FOUND})

set(SSL_ENGINE "OpenSSL" CACHE STRING "SSL Backend to use, valid options: OpenSSL, cyaSSL, GnuTLS, PolarSSL")
mark_as_advanced(SSL_ENGINE)

if (ENABLE_SSL_SUPPORT)
    hunter_add_package(OpenSSL)
    find_package(OpenSSL REQUIRED)
endif()

if (NOT BUILD_SHARED_LIBS AND NOT BUILD_STATIC_LIBS)
    message(FATAL_ERROR "One or both of BUILD_SHARED_LIBS or BUILD_STATIC_LIBS must be set to ON to build")
endif()

add_subdirectory(librabbitmq)

if (BUILD_EXAMPLES)
  add_subdirectory(examples)
endif ()

if (BUILD_TOOLS)
  if (POPT_FOUND)
    add_subdirectory(tools)
  else ()
    message(WARNING "POpt library was not found. Tools will not be built")
  endif ()
endif ()

if (BUILD_TESTS)
  if (NOT BUILD_STATIC_LIBS)
    message(FATAL_ERROR
      "Tests can only be built against static libraries "
      "(set BUILD_STATIC_LIBS=ON)")
  endif ()
  enable_testing()
  add_subdirectory(tests)
endif (BUILD_TESTS)

if (BUILD_API_DOCS)
  if (NOT DOXYGEN_FOUND)
    message(FATAL_ERROR "Doxygen is required to build the API documentation")
  endif ()
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/docs/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/docs/Doxyfile @ONLY)

  add_custom_target(docs
    COMMAND ${DOXYGEN_EXECUTABLE}
    VERBATIM
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs
    DEPENDS rabbitmq
    COMMENT "Generating API documentation"
    SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/docs/Doxyfile.in
    )
endif ()

set(libs_private ${SOCKET_LIBRARIES} ${LIBRT})
if (ENABLE_SSL_SUPPORT)
  if (SSL_ENGINE STREQUAL "OpenSSL")
    set(requires_private "openssl")
  endif()
  if (ENABLE_THREAD_SAFETY)
    set(libs_private ${libs_private} ${CMAKE_THREAD_LIBS_INIT})
  endif()
endif()

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
set(includedir "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")

configure_file(cmake/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/librabbitmq/config.h)
configure_file(librabbitmq.pc.in ${CMAKE_CURRENT_BINARY_DIR}/librabbitmq.pc @ONLY)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/librabbitmq.pc
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
  )

if (BUILD_SHARED_LIBS)
  message(STATUS "Building rabbitmq as a shared library - yes")
else ()
  message(STATUS "Building rabbitmq as a shared library - no")
endif ()

if (BUILD_STATIC_LIBS)
  message(STATUS "Building rabbitmq as a static library - yes")
else ()
  message(STATUS "Building rabbitmq as a static library - no")
endif ()

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "librabbitmqTargets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

# Note: PROJECT_VERSION is used as a VERSION
write_basic_package_version_file(
  "${version_config}" COMPATIBILITY SameMajorVersion
)

# Note: variable 'targets_export_name' used
configure_file("cmake/Config.cmake.in" "${project_config}" @ONLY)

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
)

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
)
