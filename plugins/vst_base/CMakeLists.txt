IF(LMMS_SUPPORT_VST)

INCLUDE(BuildPlugin)

IF(LMMS_BUILD_WIN32)
	ADD_DEFINITIONS(-DPTW32_STATIC_LIB)
	ADD_EXECUTABLE(RemoteVstPlugin "${CMAKE_CURRENT_SOURCE_DIR}/RemoteVstPlugin.cpp")

	IF(QT5)
		TARGET_LINK_LIBRARIES(RemoteVstPlugin Qt5::Core)
	ELSE()
		TARGET_LINK_LIBRARIES(RemoteVstPlugin -lQtCore4)
	ENDIF()
	TARGET_LINK_LIBRARIES(RemoteVstPlugin -lpthread -lgdi32 -lws2_32)
	SET_TARGET_PROPERTIES(RemoteVstPlugin PROPERTIES COMPILE_FLAGS "${COMPILE_FLAGS} -O0")
	ADD_CUSTOM_COMMAND(TARGET RemoteVstPlugin POST_BUILD COMMAND "${STRIP}" "$<TARGET_FILE:RemoteVstPlugin>")
	INSTALL(TARGETS RemoteVstPlugin RUNTIME DESTINATION "${PLUGIN_DIR}")

	# FIXME: 32-bit VST fails on MSYS
	IF(LMMS_BUILD_WIN64 AND NOT LMMS_BUILD_MSYS)
		# build 32 bit version of RemoteVstPlugin for Win64 so we can load
		# 32 bit VST plugins
		ADD_SUBDIRECTORY(Win64)
	ENDIF(LMMS_BUILD_WIN64 AND NOT LMMS_BUILD_MSYS)
ENDIF(LMMS_BUILD_WIN32)

SET(REMOTE_VST_PLUGIN_FILEPATH "RemoteVstPlugin" CACHE STRING "Relative file path to RemoteVstPlugin")

ADD_DEFINITIONS(-DREMOTE_VST_PLUGIN_FILEPATH="${REMOTE_VST_PLUGIN_FILEPATH}")
BUILD_PLUGIN(vstbase vst_base.cpp VstPlugin.cpp VstPlugin.h communication.h MOCFILES VstPlugin.h)

IF(LMMS_BUILD_LINUX AND NOT WANT_VST_NOWINE)

IF(LMMS_HOST_X86_64)
	SET(EXTRA_FLAGS -m32)

	# workaround for broken wineg++ in WINE 1.4 (shipped e.g. with Ubuntu Precise)
	EXEC_PROGRAM( ${WINE_CXX} ARGS "-v -m32 /dev/zero" OUTPUT_VARIABLE WINEBUILD_OUTPUT)
	IF("${WINEBUILD_OUTPUT}" MATCHES ".*x86_64-linux-gnu/wine/libwinecrt0.a.*")
		SET(EXTRA_FLAGS ${EXTRA_FLAGS} -nodefaultlibs /usr/lib/i386-linux-gnu/wine/libwinecrt0.a -L/usr/lib/i386-linux-gnu/wine/ -luser32 -lkernel32 -lgdi32)
	ENDIF()
	#The following check works on Fedora systems 
	IF("${WINEBUILD_OUTPUT}" MATCHES ".*lib64/wine/libwinecrt0.a.*")
		SET(EXTRA_FLAGS ${EXTRA_FLAGS} -nodefaultlibs /usr/lib/i386/wine/libwinecrt0.a -luser32 -lkernel32 -lgdi32)
	ENDIF()
ENDIF(LMMS_HOST_X86_64)

SET(WINE_CXX_FLAGS "" CACHE STRING "Extra flags passed to wineg++")

STRING(REPLACE " " ";" WINE_BUILD_FLAGS ${CMAKE_CXX_FLAGS} " " ${CMAKE_EXE_LINKER_FLAGS} " " ${WINE_CXX_FLAGS})
ADD_CUSTOM_COMMAND(
		SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/RemoteVstPlugin.cpp"
		COMMAND ${WINE_CXX}
		ARGS "-I\"${CMAKE_BINARY_DIR}\"" "-I\"${CMAKE_SOURCE_DIR}/include\"" "-I\"${CMAKE_INSTALL_PREFIX}/include/wine/windows\"" "-I\"${CMAKE_INSTALL_PREFIX}/include\"" -I/usr/include/wine/windows "\"${CMAKE_CURRENT_SOURCE_DIR}/RemoteVstPlugin.cpp\"" -ansi -mwindows -lpthread ${EXTRA_FLAGS} -fno-omit-frame-pointer ${WINE_BUILD_FLAGS} -o ../RemoteVstPlugin
		COMMAND sh -c "mv ../RemoteVstPlugin.exe ../RemoteVstPlugin || true"
		TARGET vstbase
		OUTPUTS ../RemoteVstPlugin
		)

SET_DIRECTORY_PROPERTIES(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES ../RemoteVstPlugin.exe.so)
INSTALL(PROGRAMS "${CMAKE_CURRENT_BINARY_DIR}/../RemoteVstPlugin" "${CMAKE_CURRENT_BINARY_DIR}/../RemoteVstPlugin.exe.so" DESTINATION "${PLUGIN_DIR}")
ENDIF(LMMS_BUILD_LINUX AND NOT WANT_VST_NOWINE)


ENDIF(LMMS_SUPPORT_VST)
