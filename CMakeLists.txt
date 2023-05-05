cmake_minimum_required(VERSION 3.12)

project(xschem LANGUAGES C)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(RAWTOVCD_SOURCE_FILES ${CMAKE_SOURCE_DIR}/src/rawtovcd.c)

file(GLOB_RECURSE PROJECT_SOURCE_FILES CONFIGURE_DEPENDS ${CMAKE_SOURCE_DIR}/src/*.c)
list(REMOVE_ITEM PROJECT_SOURCE_FILES ${RAWTOVCD_SOURCE_FILES})

add_executable(${PROJECT_NAME})
target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_FILES})
target_include_directories(${PROJECT_NAME} PUBLIC src/ /usr/include/cairo /usr/include/tcl8.6)
target_link_libraries(${PROJECT_NAME} ${CMAKE_DL_LIBS} tcl8.6 tk8.6 cairo xcb X11 xcb-render X11-xcb Xpm m jpeg)

add_custom_command(
    OUTPUT expandlabel.c expandlabel.h
    MAIN_DEPENDENCY ${CMAKE_SOURCE_DIR}/src/expandlabel.y
    COMMAND bison -d -o ${CMAKE_CURRENT_BINARY_DIR}/expandlabel.c ${CMAKE_SOURCE_DIR}/src/expandlabel.y
    VERBATIM
)

add_custom_command(
    OUTPUT parselabel.c
    MAIN_DEPENDENCY ${CMAKE_SOURCE_DIR}/src/parselabel.l
    COMMAND flex -l -o ${CMAKE_CURRENT_BINARY_DIR}/parselabel.c ${CMAKE_SOURCE_DIR}/src/parselabel.l
    VERBATIM
)

target_sources(${PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/expandlabel.c ${CMAKE_CURRENT_BINARY_DIR}/parselabel.c)

add_executable(rawtovcd ${RAWTOVCD_SOURCE_FILES})
