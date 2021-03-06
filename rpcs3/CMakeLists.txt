cmake_minimum_required(VERSION 2.8.12)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake_modules")
include(cotire)

project(rpcs3)

# Generate git-version.h at build time.
add_custom_target(GitVersion ALL
	DEPENDS something_that_never_exists)
add_custom_command(OUTPUT something_that_never_exists
	COMMAND ${CMAKE_COMMAND} -DSOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR}
		-P ${CMAKE_CURRENT_SOURCE_DIR}/git-version.cmake)

if(CMAKE_COMPILER_IS_GNUCXX)
	if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9.0) # TODO: the same for (apple) clang
		message(FATAL_ERROR "GCC ${CMAKE_CXX_COMPILER_VERSION} is too old.")
	endif()
	# Warnings
	add_compile_options(-Wno-attributes -Wno-enum-compare -Wno-invalid-offsetof)
elseif("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
	add_compile_options(-ftemplate-depth=1024)
	if(APPLE)
		add_compile_options(-stdlib=libc++)
	endif()
endif()

if(NOT MSVC)
	add_definitions(-DwxGUI)
	if($ENV{CI})
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O1") # fix for travis gcc OoM crash. Might be fixed with the move to containers.
	endif()
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14 -fexceptions")
	add_compile_options(-msse -msse2 -mcx16 -mssse3)
	if(WIN32)
		add_compile_options(-municode -static -mwindows)
	endif()
else()
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHa /Zc:throwingNew /D _CRT_SECURE_NO_DEPRECATE=1 /D _CRT_NON_CONFORMING_SWPRINTFS=1 /D _SCL_SECURE_NO_WARNINGS=1")
endif()

if(APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -I/opt/X11/include")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -I/opt/X11/include")
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
	#on some Linux distros shm_unlink and similar functions are in librt only
	set(ADDITIONAL_LIBS "rt" "X11")
elseif(UNIX OR NOT MSVC)
	#it seems like glibc includes the iconv functions we use but other libc
	#implementations like the one on OSX don't seem implement them
	set(ADDITIONAL_LIBS "iconv")
else()
	set(ADDITIONAL_LIBS "")
endif()

If(NOT RPCS3_SRC_DIR)
  SET(RPCS3_SRC_DIR ${CMAKE_CURRENT_LIST_DIR})
  Message("-- Initializing RPCS3_SRC_DIR=${RPCS3_SRC_DIR}")
Else()
  Message("-- Using Custom RPCS3_SRC_DIR=${RPCS3_SRC_DIR}")
EndIf()

set(CMAKE_MODULE_PATH "${RPCS3_SRC_DIR}/cmake_modules")

if(NOT WIN32)
	add_definitions(-DGL_GLEXT_PROTOTYPES)
	add_definitions(-DGLX_GLXEXT_PROTOTYPES)
endif()

find_package(wxWidgets COMPONENTS core base net aui gl xml REQUIRED)
if(NOT MSVC)
	if(APPLE)
		find_path(GLEW_INCLUDE_DIR GL/glew.h
			/usr/include/GL
			/usr/openwin/share/include
			/usr/openwin/include
			/usr/X11R6/include
			/usr/include/X11
			/opt/graphics/OpenGL/include
			/opt/graphics/OpenGL/contrib/libglew
			/usr/local/include
		)
	endif()
	find_package(GLEW REQUIRED)
	find_package(ZLIB REQUIRED)
endif()
find_package(OpenGL REQUIRED)
find_package(OpenAL REQUIRED)
find_package(LLVM CONFIG)

include("${wxWidgets_USE_FILE}")

if(APPLE)
	set(PLATFORM_ARCH "macosx/x86_64")
elseif(WIN32)
	set(PLATFORM_ARCH "Windows/x86_64")
else()
	set(PLATFORM_ARCH "linux/x86_64")
endif()

include_directories(
${GLEW_INCLUDE_DIR}
${wxWidgets_INCLUDE_DIRS}
${OPENAL_INCLUDE_DIR}
${LLVM_INCLUDE_DIRS}
"${RPCS3_SRC_DIR}/../ffmpeg/${PLATFORM_ARCH}/include"
"${RPCS3_SRC_DIR}"
"${RPCS3_SRC_DIR}/Loader"
"${RPCS3_SRC_DIR}/Crypto"
"${RPCS3_SRC_DIR}/.."
"${RPCS3_SRC_DIR}/../asmjit/src/asmjit"
"${RPCS3_SRC_DIR}/../glm"
"${RPCS3_SRC_DIR}/../libpng"
"${RPCS3_SRC_DIR}/../GSL/include"
"${RPCS3_SRC_DIR}/../rsx_program_decompiler/rsx_decompiler"
"${RPCS3_SRC_DIR}/../rsx_program_decompiler/shader_code"
"${RPCS3_SRC_DIR}/../Vulkan/Vulkan-LoaderAndValidationLayers/include"
"${RPCS3_SRC_DIR}/../Vulkan/glslang/glslang/Public"
)
if(WIN32)
	include_directories(BEFORE "${RPCS3_SRC_DIR}/../minidx9/Include")
	include_directories(BEFORE "${RPCS3_SRC_DIR}/../minidx12/Include")
endif()

if(NOT LLVM_FOUND)
	Message("LLVM not found! LLVM 3.6 is required. RPCS3 will be compiled without LLVM support.")
elseif(${LLVM_PACKAGE_VERSION} VERSION_LESS "3.6" OR ${LLVM_PACKAGE_VERSION} VERSION_EQUAL "3.7" OR ${LLVM_PACKAGE_VERSION} VERSION_GREATER "3.7")
	Message("LLVM ${LLVM_PACKAGE_VERSION} is not supported! LLVM 3.6 is required. RPCS3 will be compiled without LLVM support.")
else()
	add_definitions(${LLVM_DEFINITIONS})
	add_definitions(-DLLVM_AVAILABLE)
	if (CMAKE_BUILD_TYPE STREQUAL "Release")
		llvm_map_components_to_libnames(LLVM_LIBS mcjit vectorize x86codegen x86disassembler)
	else()
		llvm_map_components_to_libnames(LLVM_LIBS mcjit vectorize x86codegen x86disassembler mcdisassembler)
	endif()
	if(MSVC)
		set_source_files_properties(${RPCS3_SRC_DIR}/Emu/Cell/PPULLVMRecompiler.cpp PROPERTIES COMPILE_FLAGS /GR-)
	else()
		set_source_files_properties(${RPCS3_SRC_DIR}/Emu/Cell/PPULLVMRecompiler.cpp PROPERTIES COMPILE_FLAGS -fno-rtti)
	endif()
endif()

link_directories(
"${RPCS3_SRC_DIR}/../asmjit/"
"${RPCS3_SRC_DIR}/../minidx12/"
)

if(MSVC OR NOT WIN32)
	link_directories("${RPCS3_SRC_DIR}/../ffmpeg/${PLATFORM_ARCH}/lib")
endif()

get_property(dirs DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
foreach(dir ${dirs})
  message(STATUS "dir='${dir}'")
endforeach()

file(
GLOB_RECURSE
RPCS3_SRC
"${RPCS3_SRC_DIR}/rpcs3.cpp"
"${RPCS3_SRC_DIR}/config.cpp"
"${RPCS3_SRC_DIR}/stb_image.cpp"
"${RPCS3_SRC_DIR}/../Utilities/GNU.cpp"
"${RPCS3_SRC_DIR}/Emu/*"
"${RPCS3_SRC_DIR}/Gui/*"
"${RPCS3_SRC_DIR}/Loader/*"
"${RPCS3_SRC_DIR}/Crypto/*"
"${RPCS3_SRC_DIR}/../Utilities/*"
"${RPCS3_SRC_DIR}/../rsx_program_decompiler/rsx_decompiler/*"
"${RPCS3_SRC_DIR}/../rsx_program_decompiler/shader_code/*"
)

if(APPLE)
	set (EXCLUDE_DIR "/RSX/VK/")
	foreach (TMP_PATH ${RPCS3_SRC})
		string (FIND ${TMP_PATH} ${EXCLUDE_DIR} EXCLUDE_DIR_FOUND)
		if (NOT ${EXCLUDE_DIR_FOUND} EQUAL -1)
			list (REMOVE_ITEM RPCS3_SRC ${TMP_PATH})
		endif ()
	endforeach(TMP_PATH)
endif()

add_executable(rpcs3 ${RPCS3_SRC})


if(NOT MSVC)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L${CMAKE_CURRENT_BINARY_DIR}/../asmjit/") #hack because the asmjit cmake file force fno exceptions upd: not sure if vs2015 build is affected
else()
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SUBSYSTEM:WINDOWS /NODEFAULTLIB:libc.lib /NODEFAULTLIB:libcmt.lib /NODEFAULTLIB:libcd.lib /NODEFAULTLIB:libcmtd.lib /NODEFAULTLIB:msvcrtd.lib")
endif()
if(WIN32) # I'm not sure we need all of these libs, but we link them in vs
	target_link_libraries(rpcs3  odbc32.lib odbccp32.lib comctl32.lib ws2_32.lib shlwapi.lib winmm.lib kernel32.lib user32.lib gdi32.lib winspool.lib comdlg32.lib advapi32.lib shell32.lib ole32.lib oleaut32.lib uuid.lib Iphlpapi.lib)
	if(LLVM_FOUND)
		target_link_libraries(rpcs3  ${LLVM_LIBS})
	endif()
	if(NOT MSVC)
		target_link_libraries(rpcs3  ${OPENGL_LIBRARIES} ${GLEW_LIBRARY} opengl32.lib glu32.lib)
	else()
		target_link_libraries(rpcs3  dxgi.lib d2d1.lib dwrite.lib)
	endif()
	target_link_libraries(rpcs3  asmjit.lib avformat.lib avcodec.lib avutil.lib swresample.lib swscale.lib png16_static ${wxWidgets_LIBRARIES} ${OPENAL_LIBRARY} ${ADDITIONAL_LIBS} VKstatic.1 glslang OSDependent OGLCompiler SPIRV)
else()
	if(LLVM_FOUND)
		target_link_libraries(rpcs3  asmjit.a ${wxWidgets_LIBRARIES} ${OPENAL_LIBRARY} ${GLEW_LIBRARY} ${OPENGL_LIBRARIES})
		target_link_libraries(rpcs3  libavformat.a libavcodec.a libavutil.a libswresample.a libswscale.a png16_static ${ZLIB_LIBRARIES} ${LLVM_LIBS} ${ADDITIONAL_LIBS})
		if (NOT APPLE)
			target_link_libraries(rpcs3  vulkan glslang OSDependent OGLCompiler SPIRV)
		endif()
	else()
		target_link_libraries(rpcs3  asmjit.a ${wxWidgets_LIBRARIES} ${OPENAL_LIBRARY} ${GLEW_LIBRARY} ${OPENGL_LIBRARIES})
		target_link_libraries(rpcs3  libavformat.a libavcodec.a libavutil.a libswresample.a libswscale.a png16_static ${ZLIB_LIBRARIES} ${ADDITIONAL_LIBS})
		if (NOT APPLE)
			target_link_libraries(rpcs3  vulkan glslang OSDependent OGLCompiler SPIRV)
		endif()
    endif()
endif()

set_target_properties(rpcs3 PROPERTIES COTIRE_CXX_PREFIX_HEADER_INIT "${RPCS3_SRC_DIR}/stdafx.h")
cotire(rpcs3)