cmake_minimum_required (VERSION 3.5 FATAL_ERROR)

project (miniupnpc C)
set (MINIUPNPC_VERSION 2.0)
set (MINIUPNPC_API_VERSION 17)

#[[ we comment out this block as we don't support these other build types
if (NOT CMAKE_BUILD_TYPE)
  if (WIN32)
    set (DEFAULT_BUILD_TYPE MinSizeRel)
  else (WIN32)
    set (DEFAULT_BUILD_TYPE RelWithDebInfo)
  endif(WIN32)
    set (CMAKE_BUILD_TYPE ${DEFAULT_BUILD_TYPE} CACHE STRING
        "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
        FORCE)
endif()
]]

option (UPNPC_BUILD_STATIC "Build static library" TRUE)
option (UPNPC_BUILD_SHARED "Build shared library" FALSE)
option (UPNPC_BUILD_TESTS "Build test executables" FALSE)
option (NO_GETADDRINFO "Define NO_GETADDRINFO" FALSE)

mark_as_advanced (NO_GETADDRINFO)

# Interface library target for common options and flags
add_library(miniupnpc-private INTERFACE)

if (NO_GETADDRINFO)
  target_compile_definitions(miniupnpc-private INTERFACE NO_GETADDRINFO)
endif ()

if (NOT WIN32)
  target_compile_definitions(miniupnpc-private INTERFACE
    MINIUPNPC_SET_SOCKET_TIMEOUT
    _BSD_SOURCE 
    _DEFAULT_SOURCE)
  if (NOT CMAKE_SYSTEM_NAME STREQUAL "Darwin" AND NOT CMAKE_SYSTEM_NAME STREQUAL "FreeBSD" AND NOT CMAKE_SYSTEM_NAME STREQUAL "DragonFly")
    target_compile_definitions(miniupnpc-private INTERFACE _XOPEN_SOURCE=600)
    # add_definitions (-D_POSIX_C_SOURCE=200112L)
  endif ()
  if (CMAKE_SYSTEM_NAME MATCHES "(SunOS|Solaris)")
    target_compile_definitions(miniupnpc-private INTERFACE __EXTENSIONS__)
    set(CMAKE_C_STANDARD 99)
    set(CMAKE_C_STANDARD_REQUIRED ON)
  endif ()
else ()
  target_compile_definitions(miniupnpc-private INTERFACE _WIN32_WINNT=0x0501) # XP or higher for getnameinfo and friends
endif ()

if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  target_compile_definitions(miniupnpc-private INTERFACE _DARWIN_C_SOURCE)
endif ()

#[[ we comment out this block as we already set flags
# Set compiler specific build flags
if (CMAKE_COMPILER_IS_GNUCC)
  # Set our own default flags at first run.
  if (NOT CONFIGURED)

    if (NOT CMAKE_SYSTEM_NAME STREQUAL "AmigaOS")
      set (_PIC -fPIC)
    endif (CMAKE_SYSTEM_NAME STREQUAL "AmigaOS")

    set (CMAKE_C_FLAGS "${_PIC} -Wall $ENV{CFLAGS}" # CMAKE_C_FLAGS gets appended to the other C flags
        CACHE STRING "Flags used by the C compiler during normal builds." FORCE)
    set (CMAKE_C_FLAGS_DEBUG "-g -DDDEBUG"
        CACHE STRING "Flags used by the C compiler during debug builds." FORCE)
    set (CMAKE_C_FLAGS_RELEASE "-O2 -DNDEBUG"
        CACHE STRING "Flags used by the C compiler during release builds." FORCE)
    set (CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG"
        CACHE STRING "Flags used by the C compiler during release builds." FORCE)
    set (CMAKE_C_FLAGS_MINSIZEREL "-Os -DNDEBUG"
        CACHE STRING "Flags used by the C compiler during release builds." FORCE)

  endif (NOT CONFIGURED)
endif ()
]]

# always add -fPIC
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

configure_file (${CMAKE_CURRENT_SOURCE_DIR}/miniupnpcstrings.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/miniupnpcstrings.h)
target_include_directories(miniupnpc-private INTERFACE ${CMAKE_CURRENT_BINARY_DIR})

set (MINIUPNPC_SOURCES
  igd_desc_parse.c
  miniupnpc.c
  minixml.c
  minisoap.c
  minissdpc.c
  miniwget.c
  upnpcommands.c
  upnpdev.c
  upnpreplyparse.c
  upnperrors.c
  connecthostport.c
  portlistingparse.c
  receivedata.c
  connecthostport.h
  igd_desc_parse.h
  minisoap.h
  minissdpc.h
  miniupnpc.h
  miniupnpctypes.h
  miniwget.h
  minixml.h
  portlistingparse.h
  receivedata.h
  upnpcommands.h
  upnpdev.h
  upnperrors.h
  upnpreplyparse.h
)

if (NOT WIN32 AND NOT CMAKE_SYSTEM_NAME STREQUAL "AmigaOS")
  set (MINIUPNPC_SOURCES ${MINIUPNPC_SOURCES} minissdpc.c)
endif ()

if (WIN32)
  target_compile_definitions(miniupnpc-private INTERFACE MINIUPNP_STATICLIB MINIUPNP_EXPORTS)
endif ()

if (MSVC)
  target_compile_definitions(miniupnpc-private INTERFACE _CRT_SECURE_NO_WARNINGS)
endif()

if (WIN32)
  # We use set instead of find_library because otherwise static compilation on Windows breaks. Don't ask me why, just roll with it.
  # find_library (WINSOCK2_LIBRARY NAMES ws2_32 WS2_32 Ws2_32)
  # find_library (IPHLPAPI_LIBRARY NAMES iphlpapi)
  target_link_libraries(miniupnpc-private INTERFACE ws2_32 iphlpapi)
#elseif (CMAKE_SYSTEM_NAME STREQUAL "Solaris")
#  find_library (SOCKET_LIBRARY NAMES socket)
#  find_library (NSL_LIBRARY NAMES nsl)
#  find_library (RESOLV_LIBRARY NAMES resolv)
#  set (LDLIBS ${SOCKET_LIBRARY} ${NSL_LIBRARY} ${RESOLV_LIBRARY} ${LDLIBS})
endif ()

if (NOT UPNPC_BUILD_STATIC AND NOT UPNPC_BUILD_SHARED)
    message (FATAL "Both shared and static libraries are disabled!")
endif ()

if (UPNPC_BUILD_STATIC)
  add_library (libminiupnpc-static STATIC ${MINIUPNPC_SOURCES})
  set_target_properties (libminiupnpc-static PROPERTIES OUTPUT_NAME "miniupnpc")
  target_link_libraries (libminiupnpc-static PRIVATE miniupnpc-private)
  set (UPNPC_INSTALL_TARGETS ${UPNPC_INSTALL_TARGETS} libminiupnpc-static)
  set (UPNPC_LIBRARY_TARGET libminiupnpc-static)
  
  # Allows to include like #include <miniupnpc/miniupnpc.h>
  target_include_directories(libminiupnpc-static PUBLIC ../${CMAKE_CURRENT_SOURCE_DIR})
  # Suppress warnings from miniupnpc headers 
  target_include_directories(libminiupnpc-static SYSTEM PUBLIC ../${CMAKE_CURRENT_SOURCE_DIR})
  # Suppress noise warnings 
  if(MSVC)
    target_compile_options(libminiupnpc-static PRIVATE -wd4244 -wd4267)
  elseif(NOT MSVC)
    target_compile_options(libminiupnpc-static PRIVATE  -Wno-undef -Wno-unused-result -Wno-unused-value)
  endif()
  # Usage requirements from msvc/upnpc-static.vcxproj and Makefile.mingw
  if (WIN32)
    target_compile_definitions(libminiupnpc-static PUBLIC MINIUPNP_STATICLIB)
  endif()
  # add_executable (upnpc-static upnpc.c)
  # target_link_libraries (upnpc-static LINK_PRIVATE libminiupnpc-static)
endif ()

if (UPNPC_BUILD_SHARED)
  add_library (libminiupnpc-shared SHARED ${MINIUPNPC_SOURCES})
  set_target_properties (libminiupnpc-shared PROPERTIES OUTPUT_NAME "miniupnpc")
  set_target_properties (libminiupnpc-shared PROPERTIES VERSION ${MINIUPNPC_VERSION})
  set_target_properties (libminiupnpc-shared PROPERTIES SOVERSION ${MINIUPNPC_API_VERSION})
  target_link_libraries (libminiupnpc-shared PRIVATE miniupnpc-private)
  set (UPNPC_INSTALL_TARGETS ${UPNPC_INSTALL_TARGETS} libminiupnpc-shared)
  set (UPNPC_LIBRARY_TARGET libminiupnpc-shared)
  add_executable (upnpc-shared upnpc.c)
  target_link_libraries (upnpc-shared LINK_PRIVATE libminiupnpc-shared)
endif ()

if (UPNPC_BUILD_TESTS)
  add_executable (testminixml testminixml.c minixml.c igd_desc_parse.c)
  target_link_libraries (testminixml PRIVATE miniupnpc-private)

  add_executable (minixmlvalid minixmlvalid.c minixml.c)
  target_link_libraries (minixmlvalid PRIVATE miniupnpc-private)

  add_executable (testupnpreplyparse testupnpreplyparse.c
                                     minixml.c upnpreplyparse.c)
  target_link_libraries (testupnpreplyparse PRIVATE miniupnpc-private)

  add_executable (testigddescparse testigddescparse.c
                                   igd_desc_parse.c minixml.c miniupnpc.c miniwget.c minissdpc.c
                                   upnpcommands.c upnpreplyparse.c minisoap.c connecthostport.c
                                   portlistingparse.c receivedata.c
  )
  target_link_libraries (testigddescparse PRIVATE miniupnpc-private)

  add_executable (testminiwget testminiwget.c
                               miniwget.c miniupnpc.c minisoap.c upnpcommands.c minissdpc.c
                               upnpreplyparse.c minixml.c igd_desc_parse.c connecthostport.c
                               portlistingparse.c receivedata.c
  )
  target_link_libraries (testminiwget PRIVATE miniupnpc-private)

# set (UPNPC_INSTALL_TARGETS ${UPNPC_INSTALL_TARGETS} testminixml minixmlvalid testupnpreplyparse testigddescparse testminiwget)
endif ()


install (TARGETS ${UPNPC_INSTALL_TARGETS}
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib${LIB_SUFFIX}
  ARCHIVE DESTINATION lib${LIB_SUFFIX}
)
install (FILES
	miniupnpc.h
  miniwget.h
  upnpcommands.h
  igd_desc_parse.h
  upnpreplyparse.h
  upnperrors.h
  upnpdev.h
  miniupnpctypes.h
  portlistingparse.h
  miniupnpc_declspec.h
  DESTINATION include/miniupnpc
)

# commented out by Ben Boeckel, who I presume knows what he's doing;)
#set (CONFIGURED YES CACHE INTERNAL "")

# vim: ts=2:sw=2
