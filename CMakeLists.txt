cmake_minimum_required(VERSION 3.10)
project(
	zenohc 
	VERSION 0.6.0.0
	DESCRIPTION "The C bindings for Zenoh"
	HOMEPAGE_URL "https://github.com/eclipse-zenoh/zenoh-c"
	LANGUAGES C
)
set(CARGO_PROJECT_VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")
if (NOT PROJECT_VERSION_TWEAK)
	set(CARGO_PROJECT_VERSION "${CARGO_PROJECT_VERSION}-dev")
elseif(PROJECT_VERSION_TWEAK LESS 255)
	set(CARGO_PROJECT_VERSION "${CARGO_PROJECT_VERSION}-beta.${PROJECT_VERSION_TWEAK}")
endif()
message(STATUS "CARGO_PROJECT_VERSION: ${CARGO_PROJECT_VERSION}")
configure_file("${CMAKE_SOURCE_DIR}/Cargo.toml.in" "${CMAKE_SOURCE_DIR}/Cargo.toml" @ONLY)

if(NOT CMAKE_BUILD_TYPE)
	set(default_build_type "RelWithDebInfo")
	message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
	set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
		STRING "Choose the type of build." FORCE)	
	# Set the possible values of build type for cmake-gui
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
		"Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(CARGO_FLAGS "")
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
	set(CMAKE_BINARY_DIR "${CMAKE_SOURCE_DIR}/target/debug/")
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
	set(CARGO_FLAGS ${CARGO_FLAGS} "--release")
	set(CMAKE_BINARY_DIR "${CMAKE_SOURCE_DIR}/target/release/")
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
	set(CARGO_FLAGS ${CARGO_FLAGS} "--release")
	set(ENV{RUSTFLAGS} "-g")
	set(CMAKE_BINARY_DIR "${CMAKE_SOURCE_DIR}/target/release/")
elseif(CMAKE_BUILD_TYPE STREQUAL "MinSizeRel")
	set(CARGO_FLAGS ${CARGO_FLAGS} "--release")
	set(CMAKE_BINARY_DIR "${CMAKE_SOURCE_DIR}/target/release/")
else()
	message(FATAL_ERROR "Unknown CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
endif()
message(STATUS "CMAKE_BINARY_DIR: ${CMAKE_BINARY_DIR}")

execute_process(
	COMMAND cargo build ${CARGO_FLAGS}
)

add_library(zenohc_static STATIC IMPORTED)
find_library(libzenohc_static NAMES "libzenohc.a" "zenohc.lib" HINTS "${CMAKE_BINARY_DIR}" NO_DEFAULT_PATH)
message(STATUS "libzenohc_static: ${libzenohc_static}")
set_property(TARGET zenohc_static PROPERTY IMPORTED_LOCATION ${libzenohc_static})

add_library(zenohc SHARED IMPORTED)
find_library(libzenohc NAMES "libzenohc.so" "zenohc.dll" "libzenohc.dylib" HINTS "${CMAKE_BINARY_DIR}" NO_DEFAULT_PATH)
message(STATUS "libzenohc: ${libzenohc}")
set_property(TARGET zenohc PROPERTY IMPORTED_LOCATION ${libzenohc})
target_include_directories(zenohc INTERFACE "${CMAKE_SOURCE_DIR}/include")

macro(ADD_OSX_FRAMEWORK fwname target)
	find_library(FRAMEWORK_${fwname}
		NAMES ${fwname}
		PATHS ${CMAKE_OSX_SYSROOT}/System/Library
		PATH_SUFFIXES Frameworks
		NO_DEFAULT_PATH
		)
	if(${FRAMEWORK_${fwname}} STREQUAL FRAMEWORK_${fwname}-NOTFOUND)
		MESSAGE(ERROR ": Framework ${fwname} not found")
	else()
		TARGET_LINK_LIBRARIES(${target} PUBLIC "${FRAMEWORK_${fwname}}/${fwname}")
		MESSAGE(STATUS "Framework ${fwname} found at ${FRAMEWORK_${fwname}}")
	endif()
endmacro(ADD_OSX_FRAMEWORK)
	
if(APPLE OR UNIX OR WIN32)
	file(GLOB files "${CMAKE_SOURCE_DIR}/examples/*.c")
	add_custom_target(examples)
	foreach(file ${files})
		get_filename_component(target ${file} NAME_WE)
		add_executable(${target} EXCLUDE_FROM_ALL ${file})
		add_dependencies(examples ${target})
		set_property(TARGET ${target} PROPERTY C_STANDARD 11)
		set_property(TARGET ${target} PROPERTY RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/examples")
		target_include_directories(${target} PUBLIC "${CMAKE_SOURCE_DIR}/include")
		target_link_libraries(${target} PUBLIC ${libzenohc_static})
		if(APPLE)
			ADD_OSX_FRAMEWORK(Security target)
			ADD_OSX_FRAMEWORK(Foundation target)
		elseif(UNIX)
			target_link_libraries(${target} PUBLIC rt pthread m dl)
		elseif(WIN32)
			target_link_libraries(${target} PUBLIC ws2_32 crypt32 secur32 bcrypt ncrypt userenv ntdll iphlpapi runtimeobject)
		endif()
	endforeach()
else()
	message(WARNING "You platform doesn't seem to support building the examples.")
endif()


include(GNUInstallDirs)
set(CMAKE_INSTALL_INCLUDEDIR "${CMAKE_INSTALL_INCLUDEDIR}/zenohc")
install(FILES ${libzenohc} CONFIGURATIONS Release RelWithDebInfo DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(DIRECTORY "${CMAKE_SOURCE_DIR}/include/" CONFIGURATIONS Release RelWithDebInfo DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
configure_file("${CMAKE_SOURCE_DIR}/zenohc.pc.in" "${CMAKE_SOURCE_DIR}/zenohc.pc" @ONLY)
install(FILES "${CMAKE_SOURCE_DIR}/zenohc.pc" CONFIGURATIONS Release RelWithDebInfo DESTINATION "/usr/lib/pkgconfig")