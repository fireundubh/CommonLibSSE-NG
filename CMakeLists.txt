cmake_minimum_required(VERSION 3.19)

option(SKSE_SUPPORT_XBYAK "Enables trampoline support for Xbyak." OFF)
option(ENABLE_SKYRIM_SE "Enable support for Skyrim SE in the dynamic runtime feature." ON)
option(ENABLE_SKYRIM_AE "Enable support for Skyrim AE in the dynamic runtime feature." ON)
option(ENABLE_SKYRIM_VR "Enable support for Skyrim VR in the dynamic runtime feature." ON)
message("Options:")
message("\tEnable Skyrim SE: ${ENABLE_SKYRIM_SE}")
message("\tEnable Skyrim AE: ${ENABLE_SKYRIM_AE}")
message("\tEnable Skyrim VR: ${ENABLE_SKYRIM_VR}")

if(NOT ENABLE_SKYRIM_SE AND NOT ENABLE_SKYRIM_AE AND NOT ENABLE_SKYRIM_VR)
	message(FATAL_ERROR "At least one Skyrim runtime must be supported by the CommonLibSSE build.")
endif()

project(
	CommonLibSSE
	LANGUAGES CXX
	VERSION 3.1.0
)

include(GNUInstallDirs)

if("${PROJECT_SOURCE_DIR}" STREQUAL "${PROJECT_BINARY_DIR}")
	message(FATAL_ERROR "in-source builds are not allowed")
endif()

find_package(binary_io CONFIG REQUIRED)
find_package(spdlog REQUIRED CONFIG)
find_path(FAST_CPP_CSV_PARSER_INCLUDE_DIRS "csv.h")

include(cmake/sourcelist.cmake)

source_group(
	TREE "${CMAKE_CURRENT_SOURCE_DIR}"
	FILES ${SOURCES}
)

add_library(
	"${PROJECT_NAME}"
	STATIC
	${SOURCES}
	.clang-format
	CommonLibSSE.natvis
)

add_library("${PROJECT_NAME}::${PROJECT_NAME}" ALIAS "${PROJECT_NAME}")

target_compile_definitions(
	"${PROJECT_NAME}"
	PUBLIC
		WINVER=0x0601	# windows 7, minimum supported version by skyrim special edition
		_WIN32_WINNT=0x0601
		"$<$<BOOL:${SKSE_SUPPORT_XBYAK}>:SKSE_SUPPORT_XBYAK=1>"
		"$<$<BOOL:${ENABLE_SKYRIM_SE}>:ENABLE_SKYRIM_SE=1>"
		"$<$<BOOL:${ENABLE_SKYRIM_AE}>:ENABLE_SKYRIM_AE=1>"
		"$<$<BOOL:${ENABLE_SKYRIM_VR}>:ENABLE_SKYRIM_VR=1>"
)

target_compile_features(
	"${PROJECT_NAME}"
	PUBLIC
		cxx_std_20
)

if (MSVC)
	target_compile_options(
		"${PROJECT_NAME}"
		PUBLIC
			/bigobj # support large object file format
			/utf-8  # assume UTF-8 sources even without a BOM

			# warnings -> errors
			/we4715	# 'function' : not all control paths return a value

			# disable warnings
			/wd4005 # macro redefinition
			/wd4061 # enumerator 'identifier' in switch of enum 'enumeration' is not explicitly handled by a case label
			/wd4200 # nonstandard extension used : zero-sized array in struct/union
			/wd4201 # nonstandard extension used : nameless struct/union
			/wd4265 # 'type': class has virtual functions, but its non-trivial destructor is not virtual; instances of this class may not be destructed correctly
			/wd4266 # 'function' : no override available for virtual member function from base 'type'; function is hidden
			/wd4371 # 'classname': layout of class may have changed from a previous version of the compiler due to better packing of member 'member'
			/wd4514 # 'function' : unreferenced inline function has been removed
			/wd4582 # 'type': constructor is not implicitly called
			/wd4583 # 'type': destructor is not implicitly called
			/wd4623 # 'derived class' : default constructor was implicitly defined as deleted because a base class default constructor is inaccessible or deleted
			/wd4625 # 'derived class' : copy constructor was implicitly defined as deleted because a base class copy constructor is inaccessible or deleted
			/wd4626 # 'derived class' : assignment operator was implicitly defined as deleted because a base class assignment operator is inaccessible or deleted
			/wd4710 # 'function' : function not inlined
			/wd4711 # function 'function' selected for inline expansion
			/wd4820 # 'bytes' bytes padding added after construct 'member_name'
			/wd5026 # 'type': move constructor was implicitly defined as deleted
			/wd5027 # 'type': move assignment operator was implicitly defined as deleted
			/wd5045 # Compiler will insert Spectre mitigation for memory load if /Qspectre switch specified
			/wd5053 # support for 'explicit(<expr>)' in C++17 and earlier is a vendor extension
			/wd5204 # 'type-name': class has virtual functions, but its trivial destructor is not virtual; instances of objects derived from this class may not be destructed correctly
			/wd5220 # 'member': a non-static data member with a volatile qualified type no longer implies that compiler generated copy / move constructors and copy / move assignment operators are not trivial
	)
endif()
if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	target_compile_options(
			"${PROJECT_NAME}"
			PUBLIC
				-Wno-overloaded-virtual                   # function matches parent virtual function but is not an override (Clang gives false positives on this warning)
				-Wno-delete-non-abstract-non-virtual-dtor # deletion of polymorphic type which has a non-virtual destructor (necessary for SKSE proxy types, as SKSE does this)
	)
endif()

target_compile_options(${PROJECT_NAME}
		PUBLIC
		"$<$<CONFIG:RELEASE>:/O2;/Ob2;/Gw>")

target_include_directories(
	"${PROJECT_NAME}"
	PUBLIC
		"$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
		"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
)

target_include_directories(
	"${PROJECT_NAME}"
	PRIVATE
		${FAST_CPP_CSV_PARSER_INCLUDE_DIRS}
)

target_link_libraries(
	"${PROJECT_NAME}"
	PUBLIC
		binary_io::binary_io
		spdlog::spdlog
		Version.lib
)

target_precompile_headers(
	"${PROJECT_NAME}"
	PRIVATE
		include/SKSE/Impl/PCH.h
)

install(
	TARGETS "${PROJECT_NAME}"
	EXPORT "${PROJECT_NAME}-targets"
)

install(
	EXPORT "${PROJECT_NAME}-targets"
	NAMESPACE "${PROJECT_NAME}::"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)

configure_file(
	cmake/config.cmake.in
	"${PROJECT_NAME}Config.cmake"
	@ONLY
)

install(
	FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)

install(
	DIRECTORY
		"include/RE"
		"include/REL"
		"include/SKSE"
	DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)
