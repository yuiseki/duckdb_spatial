cmake_minimum_required(VERSION 3.2)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Set extension name here
set(TARGET_NAME spatial)
set(EXTENSION_NAME ${TARGET_NAME}_extension)
set(CMAKE_CXX_STANDARD 11)

project(${TARGET_NAME})

# Options

# Enable network functionality (OpenSSL and GDAL's CURL based fs/drivers)
option(SPATIAL_USE_NETWORK "Enable network functionality" ON)

include_directories(spatial/include)
add_subdirectory(spatial/src)

include_directories(spatial/third_party/ryu/include)
add_subdirectory(spatial/third_party/ryu)

include_directories(spatial/third_party/yyjson/include)
add_subdirectory(spatial/third_party/yyjson)


add_library(${EXTENSION_NAME} STATIC ${EXTENSION_SOURCES})

# Build dependencies TODO: in the future we should allow users to dynamically
# link to their own dependencies if they are building locally. If they have a
# local (compatible) GDAL installation, we should use that instead of building
# our own. (unless a static build is requested)
execute_process(
  # Generate project
  COMMAND
    ${CMAKE_COMMAND} -G ${CMAKE_GENERATOR}
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DOSX_BUILD_UNIVERSAL=${OSX_BUILD_UNIVERSAL}
    -DSPATIAL_USE_NETWORK=${SPATIAL_USE_NETWORK} -S
    ${CMAKE_CURRENT_SOURCE_DIR}/deps -B ${CMAKE_BINARY_DIR}/deps
  RESULT_VARIABLE DEPENDENCIES_GENERATE_RESULT)
if(NOT DEPENDENCIES_GENERATE_RESULT EQUAL 0)
  message(FATAL_ERROR "Could not generate dependencies project")
endif()
execute_process(
  # Build project
  COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}/deps --config
          ${CMAKE_BUILD_TYPE} RESULT_VARIABLE DEPENDENCIES_BUILD_RESULT)
if(NOT DEPENDENCIES_BUILD_RESULT EQUAL 0)
  message(FATAL_ERROR "Could not build dependencies project")
endif()

# Add dependencies
set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH}
                      ${CMAKE_BINARY_DIR}/deps/local)
set(CMAKE_FIND_LIBRARY_SUFFIXES ${CMAKE_FIND_LIBRARY_SUFFIXES} MD.lib
)# annoyingly for expat on windows
set(ZLIB_USE_STATIC_LIBS ON)
set(OPENSSL_USE_STATIC_LIBS ON)
find_library(SQLITE3_MEMVFS memvfs)
find_package(ZLIB REQUIRED)
find_package(PROJ REQUIRED)
find_package(SQLite3 REQUIRED)
find_package(GEOS REQUIRED)
find_package(GDAL REQUIRED)
find_package(EXPAT REQUIRED)
find_package(GeographicLib REQUIRED)

# Important: The link order matters, its the reverse order of dependency
target_link_libraries(
  ${EXTENSION_NAME}
  PUBLIC GDAL::GDAL
         GEOS::geos_c
         PROJ::proj
         EXPAT::EXPAT
         SQLite::SQLite3
         ZLIB::ZLIB
         ${SQLITE3_MEMVFS})

# Geographiclib is special
include_directories(${GeographicLib_INCLUDE_DIRS})
target_link_libraries(${EXTENSION_NAME} PUBLIC GeographicLib::GeographicLib)

if(SPATIAL_USE_NETWORK)
  find_package(CURL REQUIRED)
  find_package(OpenSSL REQUIRED)

  target_link_libraries(${EXTENSION_NAME} PUBLIC CURL::libcurl OpenSSL::SSL
                                                 OpenSSL::Crypto)
endif()

if(WIN32)
  target_link_libraries(${EXTENSION_NAME} PUBLIC wbemuuid.lib)
endif()

if(APPLE)
  find_library(CoreFoundation_Library CoreFoundation)
  find_library(SystemConfiguration_Library SystemConfiguration)
  target_link_libraries(${EXTENSION_NAME} PUBLIC ${CoreFoundation_Library}
                                                 ${SystemConfiguration_Library})
endif()

# Build extensions
set(PARAMETERS "-warnings")
build_loadable_extension(${TARGET_NAME} ${PARAMETERS} ${EXTENSION_SOURCES})

install(
  TARGETS ${EXTENSION_NAME}
  EXPORT "${DUCKDB_EXPORT_SET}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}")