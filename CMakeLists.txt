#####################################################################################################################
#
# CMake and System Settings
#
#####################################################################################################################

set(CMAKE_VERBOSE_MAKEFILE ON)

cmake_minimum_required(VERSION 3.24)

option(SAM_SKIP_TOOLS "Skips the lk sandbox" OFF)

if (APPLE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "12" CACHE STRING "Minimum OS X deployment version")
endif ()

if (UNIX AND NOT CMAKE_C_COMPILER)
    set(CMAKE_C_COMPILER gcc)
    set(CMAKE_CXX_COMPILER g++)
endif ()


#####################################################################################################################
#
# Project Sources
#
#####################################################################################################################

Project(lk)

set(LK_SRC
        src/absyn.cpp
        src/eval.cpp
        src/parse.cpp
        src/vm.cpp
        src/codegen.cpp
        src/invoke.cpp
        src/env.cpp
        src/lex.cpp
        src/sqlite3.c
        src/stdlib.cpp)


#####################################################################################################################
#
# Compile Options per Platform
#
#####################################################################################################################

set(CMAKE_CXX_STANDARD 11)

if (MSVC)
    add_compile_options(/W3 /wd4996 /MP)
    add_compile_definitions(WIN32 _CRT_SECURE_NO_DEPRECATE=1 _CRT_NON_CONFORMING_SWPRINTFS=1
            _SCL_SECURE_NO_WARNINGS=1 __WXMSW__ _UNICODE NOPCH LK_USE_WXWIDGETS)
    foreach (flag_var CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_DEBUG)
        set(${flag_var} "${${flag_var}} /D_DEBUG" CACHE STRING "compile flags" FORCE)
    endforeach ()
else (MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++20")
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    if (APPLE)
        add_compile_options( -fno-common)
        add_definitions(-DWX_PRECOMP)
    endif ()
    add_compile_options(-Wall -O2 )
    add_definitions(-DLK_USE_WXWIDGETS)
    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        add_compile_definitions(_DEBUG)
    else ()
        add_compile_options(-O3)
    endif ()
endif (MSVC)


#####################################################################################################################
#
# WxWidgets Package
#
#####################################################################################################################

if (UNIX)
    if(EXISTS /usr/local/bin/wx-config-3)
        set(wxWidgets_CONFIG_EXECUTABLE /usr/local/bin/wx-config-3)
        find_package(wxWidgets QUIET REQUIRED xrc stc richtext ribbon propgrid aui gl html qa adv core xml net base)
    else ()
        set(wxWidgets_CONFIG_EXECUTABLE $ENV{WXMSW3}/bin/wx-config)
        find_package(wxWidgets QUIET REQUIRED xrc stc richtext ribbon propgrid aui gl html qa adv core xml net base)
    endif ()
else ()
    set(wxWidgets_ROOT_DIR $ENV{WXMSW3})
    find_package(wxWidgets REQUIRED xrc stc richtext ribbon propgrid aui gl html qa adv core xml net base scintilla)
endif ()

if(wxWidgets_USE_FILE)
    include(${wxWidgets_USE_FILE})
endif()
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${wxWidgets_CXX_FLAGS}")


#####################################################################################################################
#
# CMake Targets
#
#####################################################################################################################

# lk library
add_library(lk STATIC ${LK_SRC})
set_target_properties(lk
        PROPERTIES
        DEBUG_POSTFIX "d"
        PREFIX ""
        )
if (MSVC)
    set_target_properties(lk
            PROPERTIES
            LINK_FLAGS /SUBSYSTEM:CONSOLE)
endif ()

target_include_directories(lk PRIVATE include)

# sandbox executable
if (NOT SAM_SKIP_TOOLS)
    add_executable(lk_sandbox ${LK_SRC} sandbox/sandbox.cpp)
    set_target_properties(lk_sandbox
            PROPERTIES
            DEBUG_POSTFIX "d"
            )
    if (MSVC)
        set_target_properties(lk_sandbox
                PROPERTIES
                LINK_FLAGS /SUBSYSTEM:WINDOWS)
    endif ()

    target_include_directories(lk_sandbox PUBLIC include)
endif()


#####################################################################################################################
#
# Link Libraries and Options
#
#####################################################################################################################

# lk library
if (${CMAKE_PROJECT_NAME} STREQUAL export_config)
    target_compile_definitions(lk PUBLIC _export_config_)
endif ()

# sandbox executable
if (NOT SAM_SKIP_TOOLS)
    if (UNIX)
        target_link_libraries(lk_sandbox -ldl)
    endif ()

#    set_property(lk_sandbox wxWidgets::wxWidgets PROPERTY MAP_IMPORTED_CONFIG_DEBUG Release)

#    string(REPLACE "d" "" wxWidgets_LIBRARIES "${wxWidgets_LIBRARIES}")

#    message(STATUS "wxWidgets Libraries before: ${wxWidgets_LIBRARIES}")
#    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
#        string(REGEX MATCHALL "optimized(.*?)l" wxWidgets_Release_LIBRARIES ${wxWidgets_LIBRARIES} )
#        string(REPLACE "optimized" "debug" wxWidgets_Release_LIBRARIES "${wxWidgets_Debug_LIBRARIES}")
#        message(STATUS "wxWidgets Release Libraries: ${wxWidgets_Release_LIBRARIES}")
#        target_link_libraries(lk_sandbox lk ${wxWidgets_Debug_LIBRARIES})
#    else()
#        target_link_libraries(lk_sandbox lk ${wxWidgets_LIBRARIES})
#    endif()      

#set(input_string "This is string1 content between string2 and then more text.")
#set(start_string "string1")
#set(end_string "string2")

# Construct the regex pattern
# Note the need to escape backslashes if you have special regex characters
# within your start_string or end_string that need to be treated literally.
#set(regex_pattern "${start_string}(.*?)${end_string}")

#string(REGEX MATCHALL "${regex_pattern}" matches ${input_string})

#foreach(match ${matches})
#    message("Matched substring: ${match}")
#endforeach()
#    string(REPLACE "debug" "optimized" wxWidgets_LIBRARIES "${wxWidgets_LIBRARIES}")

#    target_link_libraries(lk_sandbox lk wxWidgets::wxWidgets)
#    target_link_libraries(lk_sandbox lk ${wxWidgets_LIBRARIES})

#    get_target_property(DEBUG_LINK_LIBS lk_sandbox LINK_LIBRARIES)
#    message("Debug Link Libraries for lk_sandbox: $<${DEBUG_LINK_LIBS}>")
#    message("Debug Link Libraries for lk_sandbox: ${DEBUG_LINK_LIBS}")
#foreach(lib ${DEBUG_LINK_LIBS})
#    string(FIND ${lib} "NOT" INDEX)
#    string(FIND ${lib} "<" INDEX2)
#    if ((INDEX GREATER_EQUAL 0 OR INDEX2 LESS 0) AND (NOT ${lib} STREQUAL "lk"))
#        string(REPLACE "$<$<NOT:$<CONFIG:DEBUG>>:" "" lib "${lib}")
#        string(REPLACE ">" "" lib "${lib}")
#        message("lib: ${lib}")
#        list(APPEND WX_LIBS_LIST ${lib})
#    endif()
#endforeach()
#    string(REPLACE "$<$<NOT:$<CONFIG:DEBUG>>:" "" DEBUG_LINK_LIBS "${DEBUG_LINK_LIBS}")
#    string(REPLACE "$<$<CONFIG:DEBUG>>:" "" DEBUG_LINK_LIBS "${DEBUG_LINK_LIBS}")
#    string(REPLACE ">" "" DEBUG_LINK_LIBS "${DEBUG_LINK_LIBS}")
#    message("Link Libraries for lk_sandbox: ${DEBUG_LINK_LIBS}")

#message("All wxWidgets: " ${wxWidgets_LIBRARIES})
#set(FOUND "false")
#foreach(lib ${wxWidgets_LIBRARIES})
#    string(FIND ${lib} "optimized" INDEX)
#    string(FIND ${lib} "debug" INDEX2)
#    string(FIND ${lib} "wxWidgets" INDEX3)
#    if (INDEX GREATER_EQUAL 0) # skip once
#        set(FOUND "true")
#        continue()
#    elseif((INDEX2 LESS 0) AND (INDEX3 LESS 0))
#        set(FOUND "true")
#    endif()
#    if (${FOUND} STREQUAL "true")
#        set(FOUND "false")
#        message("lib: ${lib}")
#        list(APPEND WX_LIBS_LIST ${lib})
#    endif()
#endforeach()

#    list(JOIN WX_LIBS_LIST ";" WX_LIBS)
#    message("Link Libraries for lk_sandbox: ${WX_LIBS}")
    target_link_libraries(lk_sandbox lk ${wxWidgets_LIBRARIES})

endif()
