#
# Library name and options
#

# Target name
set(target metacall)

# Exit here if required dependencies are not met
message(STATUS "Lib ${target}")

# Set API export file and macro
string(TOUPPER ${target} target_upper)
set(feature_file "include/${target}/${target}_features.h")
set(export_file  "include/${target}/${target}_api.h")
set(export_macro "${target_upper}_API")

#
# Compiler warnings
#

include(Warnings)

#
# Compiler security
#

include(SecurityFlags)

#
# Configure templates
#

if(OPTION_FORK_SAFE)
	set(METACALL_FORK_SAFE 1)
endif()

if(OPTION_THREAD_SAFE)
	set(METACALL_THREAD_SAFE 1)
endif()

set(include_bin_path ${CMAKE_CURRENT_BINARY_DIR}/include/${target})

# Generate loaders plugin header
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/metacall_def.h.in ${include_bin_path}/metacall_def.h)

#
# Sources
#

set(include_path "${CMAKE_CURRENT_SOURCE_DIR}/include/${target}")
set(source_path  "${CMAKE_CURRENT_SOURCE_DIR}/source")

set(headers
	${include_bin_path}/metacall_def.h
	${include_path}/metacall.h
	${include_path}/metacall_value.h
	${include_path}/metacall_log.h
	${include_path}/metacall_allocator.h
)

set(sources
	${source_path}/metacall.c
	${source_path}/metacall_value.c
	${source_path}/metacall_log.c
	${source_path}/metacall_allocator.c
)

if(OPTION_FORK_SAFE)
	set(headers ${headers}
		${include_path}/metacall_fork.h
	)
	set(sources ${sources}
		${source_path}/metacall_fork.c
	)
endif()

# Group source files
set(header_group "Header Files (API)")
set(source_group "Source Files")
source_group_by_path(${include_path} "\\\\.h$|\\\\.hpp$"
	${header_group} ${headers})
source_group_by_path(${source_path}  "\\\\.cpp$|\\\\.c$|\\\\.h$|\\\\.hpp$"
	${source_group} ${sources})

#
# Create library
#

# Build library
add_library(${target}
	${sources}
	${headers}
)

# Create namespaced alias
add_library(${META_PROJECT_NAME}::${target} ALIAS ${target})

# Export library for downstream projects
export(TARGETS ${target} NAMESPACE ${META_PROJECT_NAME}:: FILE ${PROJECT_BINARY_DIR}/cmake/${target}/${target}-export.cmake)

# Create feature detection header
# Compilers: https://cmake.org/cmake/help/v3.1/variable/CMAKE_LANG_COMPILER_ID.html#variable:CMAKE_%3CLANG%3E_COMPILER_ID
# Feature: https://cmake.org/cmake/help/v3.1/prop_gbl/CMAKE_CXX_KNOWN_FEATURES.html

# Check for availability of module; use pre-generated version if not found
if (WriterCompilerDetectionHeaderFound)
	write_compiler_detection_header(
		FILE ${feature_file}
		PREFIX ${target_upper}
		COMPILERS AppleClang Clang GNU MSVC
		FEATURES cxx_alignas cxx_alignof cxx_constexpr cxx_final cxx_noexcept cxx_nullptr cxx_sizeof_member cxx_thread_local
		VERSION 3.2
	)
else()
	file(
		COPY ${PROJECT_SOURCE_DIR}/codegeneration/${target}_features.h
		DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/include/${target}
		USE_SOURCE_PERMISSIONS
	)
endif()

# Create API export header
generate_export_header(${target}
	EXPORT_FILE_NAME  ${export_file}
	EXPORT_MACRO_NAME ${export_macro}
)

#
# Project options
#

set_target_properties(${target}
	PROPERTIES
	${DEFAULT_PROJECT_OPTIONS}
	FOLDER "${IDE_FOLDER}"
)

#
# Include directories
#

target_include_directories(${target}
	PRIVATE
	${PROJECT_BINARY_DIR}/source/include
	${CMAKE_CURRENT_SOURCE_DIR}/include
	${CMAKE_CURRENT_BINARY_DIR}/include

	PUBLIC
	${DEFAULT_INCLUDE_DIRECTORIES}

	INTERFACE
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
	$<INSTALL_INTERFACE:include>
)

#
# Libraries
#

target_link_libraries(${target}
	PRIVATE
	${META_PROJECT_NAME}::version
	${META_PROJECT_NAME}::preprocessor
	${META_PROJECT_NAME}::environment
	${META_PROJECT_NAME}::format
	${META_PROJECT_NAME}::threading
	${META_PROJECT_NAME}::log
	${META_PROJECT_NAME}::memory
	${META_PROJECT_NAME}::portability
	${META_PROJECT_NAME}::adt
	${META_PROJECT_NAME}::filesystem
	${META_PROJECT_NAME}::dynlink
	${META_PROJECT_NAME}::detour
	${META_PROJECT_NAME}::reflect
	${META_PROJECT_NAME}::serial
	${META_PROJECT_NAME}::configuration
	${META_PROJECT_NAME}::loader

	PUBLIC
	${DEFAULT_LIBRARIES}

	INTERFACE
)

#
# Compile definitions
#

target_compile_definitions(${target}
	PRIVATE
	# Dependencies Export API
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::version,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::preprocessor,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::environment,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::format,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::threading,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::log,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::memory,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::portability,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::adt,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::filesystem,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::dynlink,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::detour,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::reflect,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::serial,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::configuration,COMPILE_DEFINITIONS>
	$<TARGET_PROPERTY:${META_PROJECT_NAME}::loader,COMPILE_DEFINITIONS>

	# MetaCall Export API
	${target_upper}_EXPORTS

	$<$<BOOL:${OPTION_FORK_SAFE}>:${target_upper}_FORK_SAFE>
	$<$<BOOL:${OPTION_THREAD_SAFE}>:${target_upper}_THREAD_SAFE>

	PUBLIC
	$<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:${target_upper}_STATIC_DEFINE>
	${DEFAULT_COMPILE_DEFINITIONS}

	INTERFACE
)

#
# Compile options
#

target_compile_options(${target}
	PRIVATE

	PUBLIC
	${DEFAULT_COMPILE_OPTIONS}

	INTERFACE
)

#
# Linker options
#

target_link_libraries(${target}
	PRIVATE

	PUBLIC
	${DEFAULT_LINKER_OPTIONS}

	INTERFACE
)

#
# Deployment
#

# Header files
install(DIRECTORY
	${CMAKE_CURRENT_SOURCE_DIR}/include/${target} DESTINATION ${INSTALL_INCLUDE}
	COMPONENT dev
)

# Generated header files
install(DIRECTORY
	${CMAKE_CURRENT_BINARY_DIR}/include/${target} DESTINATION ${INSTALL_INCLUDE}
	COMPONENT dev
)

# CMake config
install(TARGETS ${target}
	EXPORT  "${target}-export"				COMPONENT dev
	RUNTIME DESTINATION ${INSTALL_BIN}		COMPONENT runtime
	LIBRARY DESTINATION ${INSTALL_SHARED}	COMPONENT runtime
	ARCHIVE DESTINATION ${INSTALL_LIB}		COMPONENT dev
)
