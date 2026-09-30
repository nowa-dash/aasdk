cmake_minimum_required(VERSION 3.5.1)
project(aasdk VERSION 2.0.0 LANGUAGES CXX)

set(base_directory ${CMAKE_CURRENT_SOURCE_DIR})
set(sources_directory ${base_directory}/src)

set(include_directory ${base_directory}/include)
set(include_ut_directory ${base_directory}/include_ut)

include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${base_directory}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${base_directory}/lib)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${base_directory}/bin)
set(EXECUTABLE_OUTPUT_PATH ${base_directory}/bin)

SET(CMAKE_CXX_STANDARD 17)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake_modules/")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS_INIT} -fPIC -Wall -pedantic")
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0")
set(CMAKE_CXX_FLAGS_RELEASE "-g -O3")

set(Boost_USE_STATIC_LIBS OFF)
set(Boost_USE_MULTITHREADED ON)
set(Boost_USE_STATIC_RUNTIME OFF)

add_definitions(-DBOOST_ALL_DYN_LINK)

if(WIN32)
    set(WINSOCK2_LIBRARIES "ws2_32")
endif(WIN32)

if(AASDK_TEST)
    include(ExternalGtest)
endif(AASDK_TEST)

add_subdirectory(aasdk_proto)

find_package(Boost REQUIRED COMPONENTS system log OPTIONAL_COMPONENTS unit_test_framework)
find_package(libusb-1.0 REQUIRED)
find_package(Protobuf REQUIRED)
find_package(OpenSSL REQUIRED)

set(AASDK_GENERATED_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR})

include_directories(${AASDK_GENERATED_INCLUDE_DIR}
                        ${Boost_INCLUDE_DIRS}
                        ${LIBUSB_1_INCLUDE_DIRS}
                        ${PROTOBUF_INCLUDE_DIR}
                        ${OPENSSL_INCLUDE_DIR}
                        ${GTEST_INCLUDE_DIRS}
                        ${GMOCK_INCLUDE_DIRS}
                        ${include_directory}
                        ${include_ut_directory})

file(GLOB_RECURSE source_files ${sources_directory}/*.cpp)
file(GLOB_RECURSE include_files ${include_directory}/*.hpp)
file(GLOB_RECURSE tests_source_files ${sources_directory}/*.ut.cpp)
file(GLOB_RECURSE tests_include_files ${include_ut_directory}/*.hpp)

list(REMOVE_ITEM source_files ${tests_source_files})

add_library(aasdk SHARED
                ${source_files}
                ${include_files})

# Public usage requirements. Without the INSTALL_INTERFACE piece the exported
# target would carry no include directories and consumers of find_package(aasdk)
# could not resolve <f1x/aasdk/...> or <aasdk_proto/...>. libusb-1.0 is
# exposed on the install interface too, because the installed headers include
# <libusb-1.0/libusb.h> and /usr/include/libusb-1.0 is not a default search
# path.
target_include_directories(aasdk PUBLIC
                $<BUILD_INTERFACE:${include_directory}>
                $<BUILD_INTERFACE:${AASDK_GENERATED_INCLUDE_DIR}>
                $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
                $<INSTALL_INTERFACE:${LIBUSB_1_INCLUDE_DIRS}>)

add_dependencies(aasdk aasdk_proto)
# Link the imported targets rather than ${Boost_LIBRARIES} and friends so the
# export set stays relocatable. Boost::unit_test_framework is deliberately
# left out: it is only used by aasdk_ut, and putting it on the public interface
# would force every consumer of find_package(aasdk) to have gtest available.
target_link_libraries(aasdk
                        aasdk_proto
                        Boost::system
                        Boost::log
                        ${LIBUSB_1_LIBRARIES}
                        protobuf::libprotobuf
                        OpenSSL::SSL
                        OpenSSL::Crypto
                        ${WINSOCK2_LIBRARIES})

install(DIRECTORY ${include_directory}/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# Export a real CMake package so that consumers can use find_package(aasdk)
# instead of being told include/library paths on the command line.
install(TARGETS aasdk aasdk_proto
                EXPORT aasdkTargets
                LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
                ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
                RUNTIME DESTINATION ${CMAKE_INSTALL_BIN})

install(EXPORT aasdkTargets
                FILE aasdkTargets.cmake
                NAMESPACE aasdk::
                DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/aasdk)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
                "${CMAKE_CURRENT_BINARY_DIR}/aasdkConfigVersion.cmake"
                COMPATIBILITY SameMajorVersion)
configure_package_config_file(
                "${base_directory}/cmake_modules/aasdkConfig.cmake.in"
                "${CMAKE_CURRENT_BINARY_DIR}/aasdkConfig.cmake"
                INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/aasdk)

install(FILES
                "${CMAKE_CURRENT_BINARY_DIR}/aasdkConfig.cmake"
                "${CMAKE_CURRENT_BINARY_DIR}/aasdkConfigVersion.cmake"
                DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/aasdk)

if(AASDK_TEST)
    add_executable(aasdk_ut
                    ${tests_source_files}
                    ${tests_include_files})

    add_dependencies(aasdk_ut aasdk)
    target_link_libraries(aasdk_ut
                            aasdk
                            ${GMOCK_LIBRARY_PATH}
                            ${GTEST_LIBRARY_PATH})

    if(AASDK_CODE_COVERAGE)
        include(CodeCoverage)
        append_coverage_compiler_flags()
        setup_target_for_coverage(NAME aasdk_coverage EXECUTABLE aasdk_ut DEPENDENCIES aasdk_ut)
    endif(AASDK_CODE_COVERAGE)
endif(AASDK_TEST)
