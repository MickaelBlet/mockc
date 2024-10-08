set(library_project_name "${PROJECT_NAME}")

get_target_property(library_include_dirs "${library_project_name}" INTERFACE_INCLUDE_DIRECTORIES)

set(test_source_files
    "${CMAKE_CURRENT_SOURCE_DIR}/getchar.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/ioctl.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/read.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/stat.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/strcmp.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/write.cpp"
)

foreach(file ${test_source_files})
    get_filename_component(filenamewe "${file}" NAME_WE)
    add_executable("${filenamewe}.${library_project_name}.gtest" "${file}")
    set_target_properties("${filenamewe}.${library_project_name}.gtest" PROPERTIES
        CXX_STANDARD "${CMAKE_CXX_STANDARD}"
        CXX_STANDARD_REQUIRED ON
        CXX_EXTENSIONS OFF
        NO_SYSTEM_FROM_IMPORTED ON
        COMPILE_FLAGS "-pedantic -Wall -Wextra"
        INCLUDE_DIRECTORIES "${library_include_dirs};${CMAKE_CURRENT_SOURCE_DIR}/include"
        LINK_LIBRARIES "gmock_main;gtest;gmock;${library_project_name};pthread;dl"
        COMPILE_DEFINITIONS "MOCKF_DISABLE_VARIADIC_MACROS"
    )
    target_compile_options("${filenamewe}.${library_project_name}.gtest" PRIVATE -g -O0 --coverage -fprofile-arcs -ftest-coverage -fno-inline -fno-inline-small-functions -fno-default-inline)
    target_link_libraries("${filenamewe}.${library_project_name}.gtest" gcov)
    add_test(NAME "${filenamewe}.${library_project_name}.gtest" COMMAND "$<TARGET_FILE:${filenamewe}.${library_project_name}.gtest>")
endforeach()

find_program(VALGRIND "valgrind")
if (VALGRIND AND NOT TARGET "valgrind")
    set(DART_CONFIG DartConfiguration.tcl)
    add_custom_target("${DART_CONFIG}"
        COMMAND echo "MemoryCheckCommand: ${VALGRIND}" >> "${DART_CONFIG}"
        COMMENT "Generating ${DART_CONFIG}"
    )
    set(VALGRIND_ARGS
        --leak-check=full
        --error-exitcode=255
    )
    set(LOGFILE memcheck.log)
    add_custom_target("valgrind"
        COMMAND ${CMAKE_CTEST_COMMAND} -O ${LOGFILE} -D ExperimentalMemCheck --overwrite MemoryCheckCommandOptions="${VALGRIND_ARGS}" -j 16 -T memcheck -LE "^noMemcheck$"
        COMMAND tail -n1 ${LOGFILE} | grep 'Memory checking results:' > /dev/null
        COMMAND rm -f ${LOGFILE}
        DEPENDS ${DART_CONFIG}
    )
endif()