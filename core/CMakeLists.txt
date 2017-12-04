# Build libdronecore

# This finds thread libs on Linux, Mac, and Windows.
find_package(Threads REQUIRED)

include_directories(
    ${CMAKE_SOURCE_DIR}/libs/include
)

if (ANDROID)
    include_directories(curl-android-ios/prebuilt-with-ssl/android/include)
endif()

if (IOS)
    include_directories(curl-android-ios/prebuilt-with-ssl/ios/include)
endif()

if (IOS OR ANDROID OR MSVC OR APPLE)
    set(library_type "STATIC")

    # We need tinyxml2 for the camera definition parsing.
    # We use the submodule when linking statically.
    add_subdirectory(libs/tinyxml2 EXCLUDE_FROM_ALL)
    include_directories(SYSTEM libs/tinyxml2)
else()
    # Use tinyxml2 from the host system for Linux.
    set(library_type "SHARED")
endif()

if(NOT MSVC)
    set(curl_lib "curl")
    add_definitions(
        -DCURL_STATICLIB
    )
else()
    # You need to call cmake with -DWIN_CURL_INCLUDE_DIR:STRING="C:\\curl-7.54.1\\include"
    if(NOT WIN_CURL_INCLUDE_DIR)
        message(FATAL_ERROR "Please provide argument -DWIN_CURL_INCLUDE_DIR:STRING=\"path_to_curl_include\"")
    endif()
    if(NOT WIN_CURL_LIB)
        message(FATAL_ERROR "Please provide argument -DWIN_CURL_LIBSTRING=\"path_to_curl_lib\"")
    endif()
    include_directories(${WIN_CURL_INCLUDE_DIR})
    set(curl_lib ${WIN_CURL_LIB})
endif()

add_library(dronecore ${library_type}
    global_include.cpp
    connection.cpp
    device.cpp
    mavlink_parameters.cpp
    mavlink_commands.cpp
    dronecore.cpp
    dronecore_impl.cpp
    mavlink_channels.cpp
    mavlink_receiver.cpp
    serial_connection.cpp
    tcp_connection.cpp
    udp_connection.cpp
    plugin_base.cpp
    plugin_impl_base.cpp
    curl_wrapper.cpp
    http_loader.cpp
    timeout_handler.cpp
    call_every_handler.cpp
)

# cmake should check for C++11
set_property(TARGET dronecore PROPERTY CXX_STANDARD 11)
set_property(TARGET dronecore PROPERTY CXX_STANDARD_REQUIRED ON)

target_link_libraries(dronecore
    ${CMAKE_THREAD_LIBS_INIT}
    ${curl_lib}
    tinyxml2
)

# Link to Windows networking lib.
if (MSVC)
    target_link_libraries(dronecore
        ws2_32
    )
endif()

set_target_properties(dronecore
    PROPERTIES COMPILE_FLAGS ${warnings}
)

install(TARGETS dronecore
    # TODO: not sure if this is needed
    #EXPORT dronecore-targets
    DESTINATION ${lib_path}
)

install(FILES
    dronecore.h
    device.h
    ${plugin_header_paths}
    DESTINATION "include/dronecore"
)


if(NOT IOS AND NOT ANDROID)
    # SYSTEM because we don't want warnings for gtest headers.
    include_directories(SYSTEM libs/gtest/googletest/include)
    include_directories(SYSTEM libs/gtest/googlemock/include)

    add_executable(unit_tests_runner
        global_include_test.cpp
        mavlink_channels_test.cpp
        unittests_main.cpp
        # TODO: add this again
        #core/http_loader_test.cpp
        timeout_handler_test.cpp
        call_every_handler_test.cpp
        ${plugin_unittest_source_files}
    )

    if (MSVC)
        # We need this to prevent linking errors from happening in the Windows build.
        target_compile_definitions(unit_tests_runner PRIVATE -DGTEST_LINKED_AS_SHARED_LIBRARY)
        target_compile_options(unit_tests_runner PUBLIC "/wd4251" "/wd4275")
    endif()

    target_compile_definitions(unit_tests_runner PRIVATE FAKE_TIME=1)

    set_target_properties(unit_tests_runner
        PROPERTIES COMPILE_FLAGS ${warnings}
    )

    target_link_libraries(unit_tests_runner
        dronecore
        gtest
        gtest_main
        gmock
    )

    add_test(unit_tests
        unit_tests_runner
    )
endif()
