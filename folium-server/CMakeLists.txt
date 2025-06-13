cmake_minimum_required(VERSION 3.11)
project(FoliumServer VERSION 0.1.0 LANGUAGES CXX)

# Set default build type if not already set
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug)
endif()

# Set the C++ standard to C++20
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Set output directories
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

## VERSIONING ##

# Get Git commit hash for build id
find_package(Git QUIET)
if (GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD 
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_HASH 
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
else()
    set(GIT_HASH "unknown")
endif()

# Generate build date
string(TIMESTAMP BUILD_DATE "%Y-%m-%d" UTC)

# Configure version header
configure_file(
    ${CMAKE_SOURCE_DIR}/src/version.h.in 
    ${CMAKE_BINARY_DIR}/generated/version.h 
    @ONLY
)

# Include the generated directory for version info
include_directories(${CMAKE_BINARY_DIR}/generated)
include_directories(/opt/homebrew/opt/mysql-client/include)
link_directories(/opt/homebrew/opt/mysql-client/lib)

## INSTALL PACKAGES ##

# Google Test
include(FetchContent)
FetchContent_Declare(
    googletest
    URL https://github.com/google/googletest/archive/52204f78f94d7512df1f0f3bea1d47437a2c3a58.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)

# Cpp-httplib
FetchContent_Declare(
  httplib
  GIT_REPOSITORY https://github.com/yhirose/cpp-httplib.git
  GIT_TAG        v0.20.0
)

# Nlohmann/json
FetchContent_Declare(
  json
  GIT_REPOSITORY https://github.com/nlohmann/json.git
  GIT_TAG        v3.11.3
)

fetchcontent_declare(jwt-cpp 
    GIT_REPOSITORY https://github.com/Thalhammer/jwt-cpp.git
    GIT_TAG e71e0c2d584baff06925bbb3aad683f677e4d498
)
set(JWT_BUILD_EXAMPLES OFF CACHE BOOL "disable building examples" FORCE)

# Make the libraries available
FetchContent_MakeAvailable(googletest httplib json jwt-cpp)

## SET UP EXECUTABLE ##

# Build the core library
add_library(folium-core
    src/auth.cc
    src/core.cc
    src/data_access_layer.cc
    src/dispatcher.cc
    src/http_gateway.cc
    src/logger.cc
    src/pipe-filter.cc
)

# Set include directories for the library
target_include_directories(folium-core PUBLIC 
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/src 
    /usr/include/mysql-cppconn  # MySQL Connector/C++ headers
)

# Find the MySQL Connector/C++ library; adjust the path if needed
find_library(MYSQLCPPCONN_LIB 
    NAMES mysqlcppconnx mysqlcppconn
    PATHS   /usr/lib/x86_64-linux-gnu
            /opt/homebrew/opt/mysql-connector-c++/lib
            /opt/homebrew/lib
)
if(NOT MYSQLCPPCONN_LIB)
    message(FATAL_ERROR "MySQL Connector/C++ library not found")
endif()

# Find the MySQL client library
find_library(MYSQLCLIENT_LIB 
    NAMES mysqlclient
    PATHS   /usr/lib/x86_64-linux-gnu
            /usr/local/lib
            /opt/homebrew/Cellar/mysql-client/9.2.0/lib /opt/homebrew/lib
)
if(NOT MYSQLCLIENT_LIB)
    message(FATAL_ERROR "MySQL client library not found")
endif()

# Link dependencies for folium-core (notice the addition of MYSQLCLIENT_LIB)
target_link_libraries(folium-core PUBLIC 
    nlohmann_json::nlohmann_json
    httplib
    ${MYSQLCPPCONN_LIB}
    ${MYSQLCLIENT_LIB}   # New library added here
    jwt-cpp
)

# Create the server executable
add_executable(folium-server src/main.cc)
target_link_libraries(folium-server PRIVATE folium-core)

# Force rebuild on new version
add_custom_target(version-info DEPENDS ${CMAKE_BINARY_DIR}/generated/version.h)
add_dependencies(folium-server version-info)

## TESTING ##
enable_testing()

# Auth
add_executable(auth_test tests/test_auth.cc)
target_link_libraries(auth_test PRIVATE folium-core gtest gtest_main)
add_test(NAME auth_test COMMAND auth_test)

# Core
add_executable(core_test tests/test_core.cc)
target_link_libraries(core_test PRIVATE folium-core gtest gtest_main ${MYSQLCPPCONN_LIB})
add_test(NAME core_test COMMAND core_test)

# Add MySQL include and library paths
include_directories(/opt/homebrew/opt/mysql-client/include)
link_directories(/opt/homebrew/opt/mysql-client/lib)

# Link the MySQL client library
target_link_libraries(folium-core PRIVATE mysqlclient)
target_link_libraries(core_test PRIVATE folium-core gtest gtest_main mysqlclient)

# Gateway
add_executable(gateway_test tests/test_gateway.cc)
target_link_libraries(gateway_test PRIVATE folium-core gtest gtest_main httplib)
add_test(NAME gateway_test COMMAND gateway_test)

# Logging
add_executable(log_test tests/test_logger.cc)
target_link_libraries(log_test PRIVATE folium-core gtest gtest_main)
add_test(NAME log_test COMMAND log_test)

# DAL
add_executable(dal_test tests/test_dal.cc)
target_link_libraries(dal_test PRIVATE folium-core gtest gtest_main)
add_test(NAME dal_test COMMAND dal_test)

# Installation rules
install(TARGETS folium-server DESTINATION bin)
install(TARGETS folium-core 
    EXPORT folium-core-targets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
)
install(DIRECTORY include/ DESTINATION include)
