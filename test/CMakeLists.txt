option(JSON_Valgrind    "Execute test suite with Valgrind." OFF)
option(JSON_FastTests   "Skip expensive/slow tests." OFF)

# download test data
include(${CMAKE_CURRENT_SOURCE_DIR}/../cmake/download_test_data.cmake)

# test fixture to download test data
add_test(NAME "download_test_data" COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target download_test_data)
set_tests_properties(download_test_data PROPERTIES FIXTURES_SETUP TEST_DATA)

if(JSON_Valgrind)
    find_program(CMAKE_MEMORYCHECK_COMMAND valgrind)
    message(STATUS "Executing test suite with Valgrind (${CMAKE_MEMORYCHECK_COMMAND})")
    set(memcheck_command "${CMAKE_MEMORYCHECK_COMMAND} ${CMAKE_MEMORYCHECK_COMMAND_OPTIONS} --error-exitcode=1 --leak-check=full")
    separate_arguments(memcheck_command)
endif()

#############################################################################
# doctest library with the main function to speed up build
#############################################################################

add_library(doctest_main OBJECT src/unit.cpp)
set_target_properties(doctest_main PROPERTIES
    COMPILE_DEFINITIONS "$<$<CXX_COMPILER_ID:MSVC>:_SCL_SECURE_NO_WARNINGS>"
    COMPILE_OPTIONS "$<$<CXX_COMPILER_ID:MSVC>:/EHsc;$<$<CONFIG:Release>:/Od>>"
)
if (${CMAKE_VERSION} VERSION_LESS "3.8.0")
    target_compile_features(doctest_main PUBLIC cxx_range_for)
else()
    target_compile_features(doctest_main PUBLIC cxx_std_11)
endif()
target_include_directories(doctest_main PRIVATE "thirdparty/doctest")

# https://stackoverflow.com/questions/2368811/how-to-set-warning-level-in-cmake
if(MSVC)
    # Force to always compile with W4
    if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
        string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
    endif()

	# Disable warning C4566: character represented by universal-character-name '\uFF01' cannot be represented in the current code page (1252)
	# Disable warning C4996: 'nlohmann::basic_json<std::map,std::vector,std::string,bool,int64_t,uint64_t,double,std::allocator,nlohmann::adl_serializer>::operator <<': was declared deprecated
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4566 /wd4996")

	# https://github.com/nlohmann/json/issues/1114
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /bigobj")
endif()

#############################################################################
# one executable for each unit test file
#############################################################################

# check if compiler supports C++17/C++20
foreach(feature ${CMAKE_CXX_COMPILE_FEATURES})
    if (${feature} STREQUAL cxx_std_17)
        set(compiler_supports_cpp_17 TRUE)
    elseif (${feature} STREQUAL cxx_std_20)
        set(compiler_supports_cpp_20 TRUE)
    endif()
endforeach()
# Clang only supports C++17 starting from Clang 5.0
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.0)
    unset(compiler_supports_cpp_17)
endif()
# MSVC 2015 (14.0) does not support C++17
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "MSVC" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.1)
    unset(compiler_supports_cpp_17)
endif()

# Interface library for C++17 overrides
add_library(cpp17_overrides INTERFACE)

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 8.0 AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.0 AND NOT MINGW)
    # fix for https://gcc.gnu.org/bugzilla/show_bug.cgi?id=90050
    target_link_libraries(cpp17_overrides PRIVATE stdc++fs)
endif()

# Interface libraries for test-specific overrides
if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    # avoid stack overflow, see https://github.com/nlohmann/json/issues/2955
    add_library(test-cbor_overrides INTERFACE)
    target_link_options(test-cbor_overrides INTERFACE "/STACK:4000000")
    add_library(test-msgpack_overrides INTERFACE)
    target_link_options(test-msgpack_overrides INTERFACE "/STACK:4000000")
    add_library(test-ubjson_overrides INTERFACE)
    target_link_options(test-ubjson_overrides INTERFACE "/STACK:4000000")
endif()

# disable exceptions for test-disabled_exceptions
add_library(test-disabled_exceptions_overrides INTERFACE)
target_compile_definitions(test-disabled_exceptions_overrides INTERFACE JSON_NOEXCEPTION)
if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    target_compile_options(test-disabled_exceptions_overrides INTERFACE -fno-exceptions)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    # disabled due to https://github.com/nlohmann/json/discussions/2824
    #target_compile_options(test-disabled_exceptions_overrides INTERFACE /EH)
    #target_compile_definitions(test-disabled_exceptions_overrides INTERFACE _HAS_EXCEPTIONS=0)
endif()

function(add_testcases file)
    cmake_parse_arguments(add_testcases "" "CXX_STANDARD" "CXX_STANDARDS" ${ARGN})

    get_filename_component(file_basename ${file} NAME_WE)
    string(REGEX REPLACE "unit-([^$]+)" "test-\\1" testcase ${file_basename})

    if(add_testcases_CXX_STANDARD)
        if(${add_testcases_CXX_STANDARD} STREQUAL "default")
            set(testcase_exec ${testcase})
        else()
            if (compiler_supports_cpp_${add_testcases_CXX_STANDARD})
                file(READ ${file} file_content)
                string(FIND "${file_content}" JSON_HAS_CPP_${add_testcases_CXX_STANDARD} has_cpp_found)
                if(NOT ${has_cpp_found} EQUAL -1)
                    set(testcase_exec ${testcase}_cpp${add_testcases_CXX_STANDARD})
                endif()
            endif()
        endif()

        if(testcase_exec)
            add_executable(${testcase_exec} $<TARGET_OBJECTS:doctest_main> ${file})
            target_compile_definitions(${testcase_exec} PRIVATE DOCTEST_CONFIG_SUPER_FAST_ASSERTS)
            target_compile_options(${testcase_exec} PRIVATE
                $<$<CXX_COMPILER_ID:MSVC>:/EHsc;$<$<CONFIG:Release>:/Od>>
                $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wno-deprecated;-Wno-float-equal>
                $<$<CXX_COMPILER_ID:GNU>:-Wno-deprecated-declarations>
            )
            target_include_directories(${testcase_exec} PRIVATE ${CMAKE_BINARY_DIR}/include thirdparty/doctest thirdparty/fifo_map)
            target_link_libraries(${testcase_exec} PRIVATE ${NLOHMANN_JSON_TARGET_NAME})

            if(NOT ${add_testcases_CXX_STANDARD} STREQUAL "default")
                target_compile_features(${testcase_exec} PRIVATE cxx_std_${add_testcases_CXX_STANDARD})
            endif()

            if(cpp${add_testcases_CXX_STANDARD}_overrides)
                target_link_libraries(${testcase_exec} PRIVATE cpp${add_testcases_CXX_STANDARD}_overrides)
            endif()

            if(${testcase}_overrides)
                target_link_libraries(${testcase_exec} PRIVATE ${testcase}_overrides)
            endif()

            if (JSON_FastTests)
                add_test(NAME ${testcase_exec}
                    COMMAND ${testcase_exec} ${DOCTEST_TEST_FILTER}
                    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
                )
            else()
                add_test(NAME ${testcase_exec}
                    COMMAND ${testcase_exec} ${DOCTEST_TEST_FILTER} --no-skip
                    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
                )
            endif()
            set_tests_properties(${testcase_exec} PROPERTIES LABELS "all" FIXTURES_REQUIRED TEST_DATA)

            if(JSON_Valgrind)
                add_test(NAME ${testcase_exec}_valgrind
                    COMMAND ${memcheck_command} $<TARGET_FILE:${testcase_exec}> ${DOCTEST_TEST_FILTER}
                    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
                )
                set_tests_properties(${testcase_exec}_valgrind PROPERTIES LABELS "valgrind" FIXTURES_REQUIRED TEST_DATA)
            endif()
        endif()
    elseif(add_testcases_CXX_STANDARDS)
        foreach(cxx_standard ${add_testcases_CXX_STANDARDS})
            add_testcases(${file} CXX_STANDARD ${cxx_standard})
        endforeach()
    endif()
endfunction()

file(GLOB files src/unit-*.cpp)
foreach(file ${files})
    add_testcases(${file} CXX_STANDARDS default 17 20)
endforeach()

#############################################################################
# Test the generated build configs
#############################################################################

# these tests depend on the generated file nlohmann_jsonConfig.cmake
if (JSON_Install)
    add_subdirectory(cmake_import)
    add_subdirectory(cmake_import_minver)
endif()

add_subdirectory(cmake_add_subdirectory)
add_subdirectory(cmake_fetch_content)
add_subdirectory(cmake_target_include_directories)
