# cmake arguments
# CMAKE_BUILD_TYPE:			Compilation target (Debug or Release defaults to Debug)
# 
# godot-cpp cmake arguments
# GODOT_HEADERS_DIR:		This is where the gdnative include folder is (godot_source/modules/gdnative/include)
# GODOT_CUSTOM_API_FILE:	This is if you have another path for the godot_api.json
# GODOT_BIN:				Path to the godot executable to generate api.json. Leave empty to use GODOT_CUSTOM_API_FILE.
# GODOT_GENERATE_API:		Set this to OFF to avoid generating new api file with GODOT_BIN (default is ON)
#
#  scenario1: Use generated api.json with a godot executable    : default  (define GODOT_BIN to use specific godot executable or leave empty to find it in PATH)
#  scenario2: Use the api.json file in this repository          : GODOT_GENERATE_API=OFF
#  scenario3: Use externaly generated or hand crafted api.json  : GODOT_GENERATE_API=OFF GODOT_CUSTOM_API_FILE=some.json
# 
# Android cmake arguments
# CMAKE_TOOLCHAIN_FILE:		The path to the android cmake toolchain ($ANDROID_NDK/build/cmake/android.toolchain.cmake)
# ANDROID_NDK:				The path to the android ndk root folder
# ANDROID_TOOLCHAIN_NAME:	The android toolchain (arm-linux-androideabi-4.9 or aarch64-linux-android-4.9 or x86-4.9 or x86_64-4.9)
# ANDROID_PLATFORM:			The android platform version (android-23)
# More info here: https://godot.readthedocs.io/en/latest/development/compiling/compiling_for_android.html
# 
# Examples
# 
# Builds a debug version:
# cmake .
# cmake --build .
# 
# Builds a release version with clang
# CC=/usr/bin/clang CXX=/usr/bin/clang++ cmake -DCMAKE_BUILD_TYPE=Release -G "Unix Makefiles" .
# cmake --build .
# 
# Builds an android armeabi-v7a debug version:
# cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake -DANDROID_NDK=$ANDROID_NDK \
#		-DANDROID_TOOLCHAIN_NAME=arm-linux-androideabi-4.9 -DANDROID_PLATFORM=android-23 -DCMAKE_BUILD_TYPE=Debug .
# cmake --build .
# 
# Protip
# Generate the buildfiles in a sub directory to not clutter the root directory with build files:
# mkdir build && cd build && cmake -G "Unix Makefiles" .. && cmake --build .
# 
# Todo
# Test build for Windows, Mac and mingw.

project(godot-cpp)
cmake_minimum_required(VERSION 3.6)

# Default build type is Debug in the SConstruct
if("${CMAKE_BUILD_TYPE}" STREQUAL "")
	set(CMAKE_BUILD_TYPE Debug)
endif()

# Set the c++ standard to c++14
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Input from user for godot headers and the api file
set(GODOT_HEADERS_DIR "godot_headers" CACHE PATH "")
set(GODOT_CUSTOM_API_FILE "${CMAKE_CURRENT_SOURCE_DIR}/godot_headers/api.json" CACHE FILEPATH "API file used to generate bindings when GODOT_GENERATE_API=OFF")
set(GODOT_BIN "" CACHE FILEPATH "Godot executable to generate GODOT_CUSTOM_API_FILE. Leave empty to find one in the PATH.")
set(GODOT_GENERATE_API ON CACHE BOOL "Generate api.json with godot executable or not")
set(GODOT_API_TEMPLATE_GET_NODE ON CACHE BOOL "Use template version of get_node")
set(GODOT_COMPILE_FLAGS )
set(GODOT_LINKER_FLAGS )

# Define our godot-cpp library
add_library(${PROJECT_NAME})

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
	# using Visual Studio C++
  # No runtime check
	STRING(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
	string(REPLACE "/RTC1" "" CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_DEBUG})

  target_link_options(${PROJECT_NAME} PRIVATE "/WX")       # treat linker warning as error
  target_compile_options(${PROJECT_NAME} PRIVATE "/EHsc")  # Enable exception with "C" mode
  target_compile_options(${PROJECT_NAME} PRIVATE "/MP")    # Multi process build
  target_compile_options(${PROJECT_NAME} PRIVATE "/GF")    # Eliminate Duplicate strings
  target_compile_options(${PROJECT_NAME} PRIVATE "/W4")    # Warning level 4
  target_compile_options(${PROJECT_NAME} PRIVATE "/wd4201")# Disable warning for nameless structures
  # Disable conversion warning, trunkation, unreferenced var, signed missmatch
  target_compile_options(${PROJECT_NAME} PRIVATE "/wd4244" "/wd4305" "/wd4101" "/wd4018" "/wd4267")
  target_compile_definitions(${PROJECT_NAME} PRIVATE "-DWIN32_LEAN_AND_MEAN")  # avoid including old windows headers

else()

#elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
	# using Clang
#elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
	# using GCC and maybe MinGW?

	set(GODOT_LINKER_FLAGS "-static-libgcc -static-libstdc++ -Wl,-R,'$$ORIGIN'")

	# Hmm.. maybe to strikt?
	set(GODOT_COMPILE_FLAGS "-fPIC -g -Wwrite-strings")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wchar-subscripts -Wcomment -Wdisabled-optimization")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wformat -Wformat=2 -Wformat-security -Wformat-y2k")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wimport -Winit-self -Winline -Winvalid-pch -Werror")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wlong-long -Wmissing-braces -Wmissing-format-attribute")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wmissing-include-dirs -Wmissing-noreturn -Wpacked -Wpointer-arith")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wredundant-decls -Wreturn-type -Wsequence-point")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wswitch -Wswitch-enum -Wtrigraphs")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wuninitialized -Wunknown-pragmas -Wunreachable-code -Wunused-label")
	set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wunused-value -Wvariadic-macros -Wvolatile-register-var -Wno-error=attributes")

	# -Wshadow -Wextra -Wall -Weffc++ -Wfloat-equal -Wstack-protector -Wunused-parameter -Wsign-compare -Wunused-variable -Wcast-align
	# -Wunused-function -Wstrict-aliasing -Wstrict-aliasing=2 -Wmissing-field-initializers

	if(NOT CMAKE_SYSTEM_NAME STREQUAL "Android")
		set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -Wno-ignored-attributes")
	endif()

	if(CMAKE_BUILD_TYPE MATCHES Debug)
		set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -fno-omit-frame-pointer -O0")
	else()
		set(GODOT_COMPILE_FLAGS "${GODOT_COMPILE_FLAGS} -O3")
	endif(CMAKE_BUILD_TYPE MATCHES Debug)

	# Add the compile flags
	set_property(TARGET ${PROJECT_NAME} APPEND_STRING PROPERTY COMPILE_FLAGS ${GODOT_COMPILE_FLAGS})
	set_property(TARGET ${PROJECT_NAME} APPEND_STRING PROPERTY LINK_FLAGS ${GODOT_LINKER_FLAGS})

	target_compile_definitions(${PROJECT_NAME} PRIVATE $<$<CONFIG:Debug>:_DEBUG>)
	target_compile_definitions(${PROJECT_NAME} PRIVATE $<$<NOT:$<CONFIG:Debug>>:NDEBUG>)
endif()

# prepare api.json
if(NOT ${GODOT_GENERATE_API})                     # Use supplied api file
	if(NOT EXISTS ${GODOT_CUSTOM_API_FILE})
	  message(FATAL_ERROR "api.json '${GODOT_CUSTOM_API_FILE}' is not found")
  endif()
  set(GODOT_API_FILE ${GODOT_CUSTOM_API_FILE})
else()                                            # Generate api file with godot executable
  set(GODOT_API_FILE ${PROJECT_BINARY_DIR}/godot_api.json)
    if(NOT EXISTS ${GODOT_BIN})
      # find godot_exe
      find_program(GODOT_EXECUTABLE_FOUND NAMES godot godot.exe)
      if(NOT EXISTS ${GODOT_EXECUTABLE_FOUND})
        message(FATAL_ERROR "Could not find godot executable for generating api.json")
      else()
        set(GODOT_BIN ${GODOT_EXECUTABLE_FOUND})
      endif()
    endif()

	execute_process(COMMAND ${GODOT_BIN} --version OUTPUT_VARIABLE GODOT_BIN_VERSION)
	if(EXISTS ${PROJECT_BINARY_DIR}/godot_bin_version)
		file(READ ${PROJECT_BINARY_DIR}/godot_bin_version GODOT_API_VERSION)
	endif()

	if(NOT EXISTS ${GODOT_API_FILE} OR NOT "${GODOT_BIN_VERSION}" STREQUAL "${GODOT_API_VERSION}")
		message(STATUS "Generating api file: ${GODOT_API_FILE}")
		execute_process(COMMAND ${GODOT_BIN} --gdnative-generate-json-api ${GODOT_API_FILE})
		execute_process(COMMAND ${GODOT_BIN} --version OUTPUT_FILE ${PROJECT_BINARY_DIR}/godot_bin_version)
	endif()
endif()

# Generate source from the bindings file
if(${GODOT_API_FILE} IS_NEWER_THAN ${PROJECT_BINARY_DIR}/api_generated.stamp)
  message(STATUS "Generating Bindings")
  file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/binding_generator.py DESTINATION ${PROJECT_BINARY_DIR})
  file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/include/gen)
  file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/src/gen)
	execute_process(COMMAND "python" "-c" "import binding_generator; binding_generator.generate_bindings(\"${GODOT_API_FILE}\",\"${GODOT_API_TEMPLATE_GET_NODE}\")"
	WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
	RESULT_VARIABLE GENERATION_RESULT
	OUTPUT_VARIABLE GENERATION_OUTPUT)
	if (GENERATION_RESULT EQUAL "0")
    message(STATUS "Success")
    file(TOUCH ${PROJECT_BINARY_DIR}/api_generated.stamp)
  else()
    message(STATUS ${GENERATION_RESULT} ${GENERATION_OUTPUT})
    message(FATAL_ERROR "Failed")
  endif()
endif()

# Get Sources
file(GLOB_RECURSE SOURCES ${PROJECT_BINARY_DIR}/src/*.c** ${CMAKE_CURRENT_LIST_DIR}/src/*.c** )
file(GLOB_RECURSE HEADERS ${PROJECT_BINARY_DIR}/include/*.h** ${CMAKE_CURRENT_LIST_DIR}/include/*.h**)

# Define sources for our godot-cpp library
target_sources(${PROJECT_NAME} PRIVATE ${SOURCES})
target_sources(${PROJECT_NAME} PUBLIC ${HEADERS})
target_include_directories(${PROJECT_NAME}
	PUBLIC
	include
	include/core
	${PROJECT_BINARY_DIR}/include/gen
)

# Put godot headers as SYSTEM PUBLIC to exclude warnings from irrelevant headers
target_include_directories(${PROJECT_NAME}
	SYSTEM PUBLIC
	${GODOT_HEADERS_DIR}
)

add_library(godot-cpp::godot-cpp ALIAS ${PROJECT_NAME})
