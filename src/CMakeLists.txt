# Project root directory
set(ROOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/..")

# Get *.cpp and *.h source files
set(SOURCE_DIR "${ROOT_DIR}/src")
file(GLOB SOURCE_FILES "${SOURCE_DIR}/*.cpp" "${SOURCE_DIR}/*.h")

# Plugin.h
set(VERSION_HEADER "${CMAKE_CURRENT_BINARY_DIR}/src/Plugin.h")
configure_file(
	"${ROOT_DIR}/cmake/Plugin.h.in"
	"${VERSION_HEADER}"
	@ONLY
)
source_group("src" FILES "${VERSION_HEADER}")

# version.rc
configure_file(
	"${ROOT_DIR}/cmake/version.rc.in"
	"${CMAKE_CURRENT_BINARY_DIR}/version.rc"
	@ONLY
)

# Setup SKSE project library
add_library(
	"${PROJECT_NAME}"
	SHARED
	${SOURCE_FILES}
	"${VERSION_HEADER}"
	"${CMAKE_CURRENT_BINARY_DIR}/version.rc"
	"${ROOT_DIR}/.clang-format"
	"${ROOT_DIR}/.editorconfig"
)

# Compile with C++20 features
target_compile_features(
	"${PROJECT_NAME}"
	PRIVATE
		cxx_std_23
)

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
	target_compile_options(
		"${PROJECT_NAME}"
		PRIVATE
			"/sdl"	# Enable Additional Security Checks
			"/utf-8"	# Set Source and Executable character sets to UTF-8
			"/Zi"	# Debug Information Format

			"/permissive-"	# Standards conformance
			"/Zc:preprocessor"	# Enable preprocessor conformance mode

			"/wd4200" # nonstandard extension used : zero-sized array in struct/union

			"$<$<CONFIG:DEBUG>:>"
			"$<$<CONFIG:RELEASE>:/Zc:inline;/JMC-;/Ob3>"
	)

	target_link_options(
		"${PROJECT_NAME}"
		PRIVATE
			"$<$<CONFIG:DEBUG>:/INCREMENTAL;/OPT:NOREF;/OPT:NOICF>"
			"$<$<CONFIG:RELEASE>:/INCREMENTAL:NO;/OPT:REF;/OPT:ICF;/DEBUG:FULL>"
	)
endif()

# Default output directory for compiled files
target_include_directories(
	"${PROJECT_NAME}"
	PRIVATE
		"${CMAKE_CURRENT_BINARY_DIR}/src"
		"${SOURCE_DIR}"
)

# CommonLib
if(${SKYRIM_AE})
	add_subdirectory("${ROOT_DIR}/CommonLib/AE" CommonLibSSE EXCLUDE_FROM_ALL)
    target_link_libraries(
        "${PROJECT_NAME}"
        PUBLIC
            CommonLibSSE::CommonLibSSE
    )
elseif(${SKYRIM_SE})
	add_subdirectory("${ROOT_DIR}/CommonLib/SE" CommonLibSSE EXCLUDE_FROM_ALL)
    target_link_libraries(
        "${PROJECT_NAME}"
        PUBLIC
            CommonLibSSE::CommonLibSSE
    )
elseif(${SKYRIM_VR})
	add_subdirectory("${ROOT_DIR}/CommonLib/VR" CommonLibVR EXCLUDE_FROM_ALL)
    target_link_libraries(
        "${PROJECT_NAME}"
        PUBLIC
            CommonLibVR::CommonLibVR
    )
endif()

# Include spdlog
find_package(spdlog CONFIG REQUIRED)
target_link_libraries("${PROJECT_NAME}" PUBLIC spdlog::spdlog)

# Precompiled Headers (PCH)
target_precompile_headers(
	"${PROJECT_NAME}"
	PRIVATE
		"${SOURCE_DIR}/PCH.h"
)

# Get output directory based on configured version of Skyrim
if(${SKYRIM_AE})
	if(DEFINED ENV{SKSE_SKYRIM_AE_DEV})
		set(MOD_FOLDER $ENV{SKSE_SKYRIM_AE_DEV})
	elseif(DEFINED ENV{SKYRIM_MODS})
		set(MOD_FOLDER "$ENV{SKYRIM_MODS}/${PROJECT_NAME} - ${SKYRIM_VERSION}")
	else()
		set(MOD_FOLDER "${ROOT_DIR}/output/${SKYRIM_VERSION}")
	endif()
elseif(${SKYRIM_SE})
	if(DEFINED ENV{SKSE_SKYRIM_SE_DEV})
		set(MOD_FOLDER $ENV{SKSE_SKYRIM_SE_DEV})
	elseif(DEFINED ENV{SKYRIM_MODS})
		set(MOD_FOLDER "$ENV{SKYRIM_MODS}/${PROJECT_NAME} - ${SKYRIM_VERSION}")
	else()
		set(MOD_FOLDER "${ROOT_DIR}/output/${SKYRIM_VERSION}")
	endif()
elseif(${SKYRIM_VR})
	if(DEFINED ENV{SKSE_SKYRIM_VR_DEV})
		set(MOD_FOLDER $ENV{SKSE_SKYRIM_VR_DEV})
	elseif(DEFINED ENV{SKYRIM_MODS})
		set(MOD_FOLDER "$ENV{SKYRIM_MODS}/${PROJECT_NAME} - ${SKYRIM_VERSION}")
	else()
		set(MOD_FOLDER "${ROOT_DIR}/output/${SKYRIM_VERSION}")
	endif()
endif()

set(DLL_FOLDER "${MOD_FOLDER}/SKSE/Plugins")

# Create output directory if it does not exist
add_custom_command(
	TARGET "${PROJECT_NAME}"
	POST_BUILD
	COMMAND "${CMAKE_COMMAND}" -E make_directory "${DLL_FOLDER}"
	VERBATIM
)

file(GLOB PLUGIN_FILES "${ROOT_DIR}/*.esp" "${ROOT_DIR}/*.esm" "${ROOT_DIR}/*.esl")

message("PLUGIN FILES ${PLUGIN_FILES}")

# Print the paths that files will be output to when the project is built
message("Build will output .dll and .pdb to ${DLL_FOLDER}")
if(EXISTS "${ROOT_DIR}/Scripts")
	message("Build will output Papyrus scripts to ${MOD_FOLDER}")
endif()
if(PLUGIN_FILES)
	message("Plugin files will be output to ${MOD_FOLDER}")
endif()

# Copy the output .dll and debug .pdb to the configured output directory
add_custom_command(
	TARGET "${PROJECT_NAME}"
	POST_BUILD
	COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}"
	COMMAND "${CMAKE_COMMAND}" -E copy_if_different "$<TARGET_PDB_FILE:${PROJECT_NAME}>" "${DLL_FOLDER}"
	COMMAND "${CMAKE_COMMAND}" -E copy_directory "${ROOT_DIR}/Scripts" "${MOD_FOLDER}/Scripts"
	COMMAND "${CMAKE_COMMAND}" -E copy_directory "${ROOT_DIR}/Source" "${MOD_FOLDER}/Source"
	VERBATIM
)

# Copy Scripts/ and/or Source/ if they are present (at prepare time)
if(EXISTS "${ROOT_DIR}/Scripts")
	add_custom_command(
		TARGET "${PROJECT_NAME}"
		POST_BUILD
		COMMAND "${CMAKE_COMMAND}" -E copy_directory "${ROOT_DIR}/Scripts" "${MOD_FOLDER}/Scripts"
		VERBATIM
	)
endif()
if(EXISTS "${ROOT_DIR}/Source")
	add_custom_command(
		TARGET "${PROJECT_NAME}"
		POST_BUILD
		COMMAND "${CMAKE_COMMAND}" -E copy_directory "${ROOT_DIR}/Source" "${MOD_FOLDER}/Source"
		VERBATIM
	)
endif()

# Copy .esp .esm .esl files, if present (at prepare time)
if(PLUGIN_FILES)
	add_custom_command(
		TARGET "${PROJECT_NAME}"
		POST_BUILD
		COMMAND "${CMAKE_COMMAND}" -E copy_if_different ${PLUGIN_FILES} "${MOD_FOLDER}"
		VERBATIM
	)
endif()
