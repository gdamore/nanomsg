#
#   Copyright (c) 2012 Martin Sustrik  All rights reserved.
#   Copyright (c) 2013 GoPivotal, Inc.  All rights reserved.
#   Copyright 2015 Garrett D'Amore <garrett@damore.org>
#
#   Permission is hereby granted, free of charge, to any person obtaining a copy
#   of this software and associated documentation files (the "Software"),
#   to deal in the Software without restriction, including without limitation
#   the rights to use, copy, modify, merge, publish, distribute, sublicense,
#   and/or sell copies of the Software, and to permit persons to whom
#   the Software is furnished to do so, subject to the following conditions:
#
#   The above copyright notice and this permission notice shall be included
#   in all copies or substantial portions of the Software.
#
#   THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#   IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#   FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
#   THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#   LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
#   FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
#   IN THE SOFTWARE.
#

cmake_minimum_required (VERSION 2.8.12)
include (CheckIncludeFiles)
include (CheckLibraryExists)
include (CheckFunctionExists)
include (CheckSymbolExists)
include (CheckCSourceCompiles)
include (CheckStructHasMember)
include (GNUInstallDirs)

project (nanomsg LANGUAGES C)
enable_testing ()

# Determine our ABI version
file (READ src/nn.h NN_H)

string (REGEX MATCH "NN_VERSION_CURRENT ([0-9]+)" TMP0 ${NN_H})
string (REGEX REPLACE "NN_VERSION_CURRENT +" "" NN_VERSION_CURRENT ${TMP0})

string (REGEX MATCH "NN_VERSION_REVISION ([0-9]+)" TMP0 ${NN_H})
string (REGEX REPLACE "NN_VERSION_REVISION +" "" NN_VERSION_REVISION ${TMP0})

string (REGEX MATCH "NN_VERSION_AGE ([0-9]+)" TMP0 ${NN_H})
string (REGEX REPLACE "NN_VERSION_AGE +" "" NN_VERSION_AGE ${TMP0})
set (NN_ABI_VERSION ${NN_VERSION_CURRENT}.${NN_VERSION_REVISION}.${NN_VERSION_AGE})

set (CMAKE_SKIP_BUILD_RPATH FALSE)
set (CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
#set (CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set (CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
list (FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
if ("${isSystemDir}" STREQUAL "-1")
	set (CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
endif ()

#  Platform checks.

if (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    add_definitions (-DNN_HAVE_WINDOWS)
    add_definitions (-D_CRT_SECURE_NO_WARNINGS)

    if (MINGW)
        add_definitions (-DNN_HAVE_MINGW -D_WIN32_WINNT=0x0600)
    endif ()
elseif (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    add_definitions (-DNN_HAVE_OSX)
elseif (${CMAKE_SYSTEM_NAME} MATCHES "Linux" )
    add_definitions (-DNN_HAVE_LINUX)
elseif (${CMAKE_SYSTEM_NAME} MATCHES "SunOS" )
    add_definitions (-DNN_HAVE_SOLARIS)
endif ()

# Should be harmless on everything that doesn't use it.  Needed on some
# platforms.
add_definitions (-D_REENTRANT)

# Feature checks.
macro (NN_CHECK_INC HEADER DEF)
	check_include_files (${HEADER} ${DEF})
	if (${DEF})
		add_definitions (-D${DEF}=1)
	endif ()
endmacro (NN_CHECK_INC)

macro (NN_CHECK_LIB LIB SYM DEF)
	check_library_exists (${LIB} ${SYM} "" ${DEF})
	if (${DEF})
		add_definitions (-D${DEF}=1)
		set (EXTRA_LIBS ${EXTRA_LIBS} ${LIB})
		# add_link_libraries
	endif ()
endmacro (NN_CHECK_LIB)

macro (NN_CHECK_FUNC NAME DEF)
	set (CMAKE_REQUIRED_LIBRARIES ${EXTRA_LIBS})
	check_function_exists (${NAME} ${DEF})
	
	if (${DEF})
		add_definitions (-D${DEF}=1)
	endif ()
endmacro (NN_CHECK_FUNC)

macro (NN_CHECK_SYM SYM HEADER DEF)
	check_symbol_exists (${SYM} ${HEADER} ${DEF})
	if (${DEF})
		add_definitions (-D${DEF}=1)
	endif ()
endmacro (NN_CHECK_SYM)

macro (NN_CHECK_MEMBER STR MEM HEADER DEF)
	check_struct_has_member (${STR} ${MEM} ${HEADER} ${DEF})
	if (${DEF})
		add_definitions (-D${DEF}=1)
	endif ()
endmacro (NN_CHECK_MEMBER)


nn_check_inc (netinet/in.h		NN_HAVE_INET_IN)
nn_check_inc (netdb.h			NN_HAVE_NETDB)
nn_check_inc (unistd.h			NN_HAVE_UNISTD)
nn_check_inc (sys/socket.h		NN_HAVE_SYS_SOCKET)
nn_check_inc (sys/ioctl.h		NN_HAVE_SYS_IOCTL)
nn_check_inc (stdint.h			NN_HAVE_STDINT)
nn_check_inc (inttypes.h		NN_HAVE_INTTYPES)

nn_check_lib (socket socket 		NN_HAVE_LIBSOCKET)
nn_check_lib (nsl gethostent 		NN_HAVE_LIBNSL)
nn_check_lib (pthread pthread_create	NN_HAVE_PTHREAD)
nn_check_lib (rt clock_gettime		NN_HAVE_RT)
nn_check_lib (anl getaddrinfo_a 	NN_HAVE_GETADDRINFO_A)

nn_check_func (eventfd			NN_HAVE_EVENTFD)
nn_check_func (pipe			NN_HAVE_PIPE)
nn_check_func (pipe2			NN_HAVE_PIPE2)
nn_check_func (gethrtime		NN_HAVE_GETHRTIME)

nn_check_func (poll			NN_HAVE_POLL)
nn_check_func (epoll_create		NN_HAVE_EPOLL)
nn_check_func (kqueue			NN_HAVE_KQUEUE)

nn_check_func (socketpair		NN_HAVE_SOCKETPAIR)
nn_check_func (sem_wait			NN_HAVE_SEMAPHORE)
nn_check_func (accept4			NN_HAVE_ACCEPT4)
nn_check_func (getifaddrs		NN_HAVE_GETIFADDRS)

nn_check_func(atomic_add_32_nv 		NN_ATOMIC_SOLARIS)
nn_check_func(__sync_fetch_and_sub	NN_ATOMIC_GCC_BUILTINS)

nn_check_sym (CLOCK_MONOTONIC time.h	NN_HAVE_CLOCK_MONTONIC)
nn_check_sym (SIOCGIFADDR sys/ioctl.h	NN_HAVE_SIOCGIFADDR)

nn_check_member ("struct msghdr" msg_control sys/socket.h NN_HAVE_MSG_CONTROL)

# Select the polling mechanism
if (${NN_HAVE_EPOLL})
	add_definitions (-DNN_USE_EPOLL=1)
elseif (${NN_HAVE_KQUEUE})
	add_definitions (-DNN_USE_KQUEUE=1)
elseif (${NN_HAVE_POLL})
	add_definitions (-DNN_USE_POLL=1)
else ()
    message (WARNING "ERROR: No suitable polling mechanism found.")
endif ()

# Select the interface mechanism
if (${NN_HAVE_GETIFADDRS})
	add_definitions (-DNN_USE_IFADDRS=1)
elseif (${NN_HAVE_SIOCGIFADDR})
	add_definitions (-DNN_USE_SIOCGIFADDR=1)
else ()
	add_definitions (-DNN_USE_LITERAL_IFADDR=1)
endif ()

# Selct event mechanism
if (${NN_HAVE_EVENTFD})
	add_definitions (-DNN_USE_EVENTFD=1)
elseif (${NN_HAVE_PIPE})
	add_definitions (-DNN_USE_PIPE=1)
elseif (${NN_HAVE_SOCKETPAIR})
	add_definitions (-DNN_USE_SOCKETPAIR=1)
endif ()

#  Build the library itself.

add_subdirectory (src)

#  Build the tools

add_executable (nanocat tools/nanocat.c tools/options.c)
target_link_libraries (nanocat nanomsg)

#  Build unit tests.

set (all_tests "")

macro (add_libnanomsg_test NAME)
    list (APPEND all_tests ${NAME})
    add_executable (${NAME} tests/${NAME}.c)
    target_link_libraries (${NAME} nanomsg)
    add_test (NAME ${NAME} COMMAND ${NAME})
endmacro (add_libnanomsg_test)

#  Transport tests.
add_libnanomsg_test (inproc)
add_libnanomsg_test (inproc_shutdown)
add_libnanomsg_test (ipc)
add_libnanomsg_test (ipc_shutdown)
add_libnanomsg_test (ipc_stress)
add_libnanomsg_test (tcp)
add_libnanomsg_test (tcp_shutdown)
add_libnanomsg_test (ws)
add_libnanomsg_test (tcpmux)

#  Protocol tests.
add_libnanomsg_test (pair)
add_libnanomsg_test (pubsub)
add_libnanomsg_test (reqrep)
add_libnanomsg_test (pipeline)
add_libnanomsg_test (survey)
add_libnanomsg_test (bus)

#  Feature tests.
add_libnanomsg_test (block)
add_libnanomsg_test (term)
add_libnanomsg_test (timeo)
add_libnanomsg_test (iovec)
add_libnanomsg_test (msg)
add_libnanomsg_test (prio)
add_libnanomsg_test (poll)
add_libnanomsg_test (device)
add_libnanomsg_test (emfile)
add_libnanomsg_test (domain)
add_libnanomsg_test (trie)
add_libnanomsg_test (list)
add_libnanomsg_test (hash)
add_libnanomsg_test (symbol)
add_libnanomsg_test (separation)
add_libnanomsg_test (zerocopy)
add_libnanomsg_test (shutdown)
add_libnanomsg_test (cmsg)

#  Build the performance tests.

macro (add_libnanomsg_perf NAME)
    add_executable (${NAME} perf/${NAME}.c)
    target_link_libraries (${NAME} nanomsg)
endmacro (add_libnanomsg_perf)

add_libnanomsg_perf (inproc_lat)
add_libnanomsg_perf (inproc_thr)
add_libnanomsg_perf (local_lat)
add_libnanomsg_perf (remote_lat)
add_libnanomsg_perf (local_thr)
add_libnanomsg_perf (remote_thr)

#  NSIS package

install (FILES src/nn.h DESTINATION include/nanomsg)
install (FILES src/inproc.h DESTINATION include/nanomsg)
install (FILES src/ipc.h DESTINATION include/nanomsg)
install (FILES src/tcp.h DESTINATION include/nanomsg)
install (FILES src/ws.h DESTINATION include/nanomsg)
install (FILES src/pair.h DESTINATION include/nanomsg)
install (FILES src/pubsub.h DESTINATION include/nanomsg)
install (FILES src/reqrep.h DESTINATION include/nanomsg)
install (FILES src/pipeline.h DESTINATION include/nanomsg)
install (FILES src/survey.h DESTINATION include/nanomsg)
install (FILES src/bus.h DESTINATION include/nanomsg)

install (TARGETS nanocat RUNTIME DESTINATION bin)

configure_file(libnanomsg.pc.in libnanomsg.pc)
install(FILES ${CMAKE_BINARY_DIR}/libnanomsg.pc DESTINATION lib/pkgconfig)

set (CPACK_GENERATOR "NSIS")
set (CPACK_PACKAGE_NAME "nanomsg")
include (CPack)
