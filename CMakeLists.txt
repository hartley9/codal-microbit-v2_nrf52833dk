project(codal-microbit-v2)

# Actually merge our multiple configs into a single set of values so we can query them easily.
# Note: This will be removed when we move to a library-ified version of the build system
#       so do not rely on it for anything.
#
# Note: This form of loop isn't available until we move to at least CMake 3.17+, so the second form
#       below has been added to do the same, in a more verbose way.
#foreach( _key _val IN ZIP_LISTS FINAL_FIELDS FINAL_VALUES )
#    set( "__merged.config.${_key}" "${_val}" )
#endforeach()

# Grab the latest tag from git
execute_process( WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} COMMAND git describe --tags --abbrev=0 OUTPUT_VARIABLE git_tag_string )
message( "${CMAKE_CURRENT_SOURCE_DIR}" )
message( "${git_tag_string}" )
string( REGEX MATCHALL "[0-9]+" git_tag "${git_tag_string}" )

# If we had a valid git response, update the values
set( version_major 0 )
set( version_minor 0 )
set( version_patch 0 )
list(LENGTH git_tag len)
if( ${len} GREATER 0 )
    list(GET git_tag 0 version_major )
    list(GET git_tag 1 version_minor )
    list(GET git_tag 2 version_patch )
endif()

# Write out the version values to our configuration file - CODAL_VERSION_STRING is generated in the preprocessor from these.
file( APPEND "${EXTRA_INCLUDES_PATH}" " #define CODAL_VERSION_MAJOR ${version_major}\n" )
file( APPEND "${EXTRA_INCLUDES_PATH}" " #define CODAL_VERSION_MINOR ${version_minor}\n" )
file( APPEND "${EXTRA_INCLUDES_PATH}" " #define CODAL_VERSION_PATCH ${version_patch}\n" )

LIST( LENGTH FINAL_FIELDS _idxMax )
MATH( EXPR _idxMax "${_idxMax} - 1" )
foreach( _index RANGE ${_idxMax} )
    LIST( GET FINAL_FIELDS ${_index} _key )
    LIST( GET FINAL_VALUES ${_index} _val )
    set( "__merged.config.${_key}" "${_val}" )
endforeach()

if( DEFINED __merged.config.SOFTDEVICE_PRESENT AND NOT "${__merged.config.DEVICE_BLE}" STREQUAL "1" )
    message( FATAL_ERROR "Do not define SOFTDEVICE_PRESENT in your configuration, use DEVICE_BLE instead." )
endif()

# Determine if we need SOFTDEVICE support by checking if BLE is enabled.
if( "${__merged.config.DEVICE_BLE}" STREQUAL "1" )
    message( "Building WITH softdevice support" )
    file( APPEND "${EXTRA_INCLUDES_PATH}" " #define SOFTDEVICE_PRESENT    1" )
    set(CMAKE_LINKER_FLAGS "${CMAKE_LINKER_FLAGS} -T\"${CMAKE_CURRENT_LIST_DIR}/ld/nrf52833-softdevice.ld\"" PARENT_SCOPE)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -T\"${CMAKE_CURRENT_LIST_DIR}/ld/nrf52833-softdevice.ld\"" PARENT_SCOPE)
else()
    message( "Building WITHOUT softdevice support" )
    set(CMAKE_LINKER_FLAGS "${CMAKE_LINKER_FLAGS} -T\"${CMAKE_CURRENT_LIST_DIR}/ld/nrf52833.ld\"" PARENT_SCOPE)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -T\"${CMAKE_CURRENT_LIST_DIR}/ld/nrf52833.ld\"" PARENT_SCOPE)
endif()

# find sources and headers
RECURSIVE_FIND_DIR(INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/inc" "*.h")
RECURSIVE_FIND_FILE(SOURCE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/source" "*.c??")

RECURSIVE_FIND_DIR(INCLUDE_DIRS2 "${CMAKE_CURRENT_SOURCE_DIR}/model" "*.h")
RECURSIVE_FIND_FILE(SOURCE_FILES2 "${CMAKE_CURRENT_SOURCE_DIR}/model" "*.c??")

list(APPEND INCLUDE_DIRS "${INCLUDE_DIRS2}")
list(APPEND SOURCE_FILES "${SOURCE_FILES2}")

# find prebuilt libraries and objects
RECURSIVE_FIND_FILE(LIB_OBJECT_FILES "${CMAKE_CURRENT_LIST_DIR}/lib" "*.o")
RECURSIVE_FIND_FILE(LIB_ARCHIVE_FILES "${CMAKE_CURRENT_LIST_DIR}/lib" "*.a")

set(CMAKE_SYSTEM_PROCESSOR "armv7-m" PARENT_SCOPE)

# add them
include_directories(${INCLUDE_DIRS})

# create our target
add_library(codal-microbit-v2 ${SOURCE_FILES})

target_link_libraries(
    codal-microbit-v2
    codal-nrf52
    codal-core
    codal-microbit-nrf5sdk
    ${LIB_OBJECT_FILES}
    ${LIB_ARCHIVE_FILES}
)

# expose it to parent cmake.
target_include_directories(codal-microbit-v2 PUBLIC ${INCLUDE_DIRS})