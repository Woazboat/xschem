cmake_minimum_required(VERSION 3.12)

project(xschem LANGUAGES C)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(RAWTOVCD_SOURCE_FILES "${CMAKE_SOURCE_DIR}/src/rawtovcd.c")

file(GLOB_RECURSE PROJECT_SOURCE_FILES CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/src/*.c")
list(REMOVE_ITEM PROJECT_SOURCE_FILES ${RAWTOVCD_SOURCE_FILES})

add_executable(${PROJECT_NAME})
target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCE_FILES})
target_include_directories(${PROJECT_NAME} PUBLIC src/ /usr/include/cairo /usr/include/tcl8.6)
target_link_libraries(${PROJECT_NAME} ${CMAKE_DL_LIBS} tcl8.6 tk8.6 cairo xcb X11 xcb-render X11-xcb Xpm m jpeg)

add_custom_command(
    OUTPUT expandlabel.c expandlabel.h
    MAIN_DEPENDENCY "${CMAKE_SOURCE_DIR}/src/expandlabel.y"
    COMMAND bison -d -o "${CMAKE_CURRENT_BINARY_DIR}/expandlabel.c" "${CMAKE_SOURCE_DIR}/src/expandlabel.y"
    VERBATIM
)

add_custom_command(
    OUTPUT parselabel.c
    MAIN_DEPENDENCY "${CMAKE_SOURCE_DIR}/src/parselabel.l"
    COMMAND flex -l -o "${CMAKE_CURRENT_BINARY_DIR}/parselabel.c" "${CMAKE_SOURCE_DIR}/src/parselabel.l"
    VERBATIM
)

target_sources(${PROJECT_NAME} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/expandlabel.c" "${CMAKE_CURRENT_BINARY_DIR}/parselabel.c")

target_compile_options(${PROJECT_NAME} PUBLIC
  $<$<CONFIG:Release>:-O2>
  $<$<CONFIG:RelWithDebInfo>:-g -O2>
  $<$<CONFIG:Debug>:-g -O0 -Wconversion -Wno-sign-conversion>
  $<$<CONFIG:Profile>:-pg -O0>
)

add_executable(rawtovcd ${RAWTOVCD_SOURCE_FILES})

target_compile_options(rawtovcd PUBLIC
  $<$<CONFIG:Release>:-O2>
  $<$<CONFIG:RelWithDebInfo>:-g -O2>
  $<$<CONFIG:Debug>:-g -O0>
  $<$<CONFIG:Profile>:-pg -O0>
)

set(USER_CONF_DIR "~/.xschem")
set(USER_LIB_DIR "${USER_CONF_DIR}/xschem_library")

#######################
# Install configuration
#######################

include(GNUInstallDirs)

install(TARGETS "${PROJECT_NAME}" rawtovcd)

install(DIRECTORY doc/
        TYPE DOC
        FILES_MATCHING
        PATTERN "*.svg"
        PATTERN "*.html"
        PATTERN "*.css"
        PATTERN "*.png"
        PATTERN "manpages" EXCLUDE)

install(DIRECTORY xschem_library/devices
        DESTINATION "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/xschem_library"
        FILES_MATCHING
        PATTERN "*.sym"
        PATTERN "*.sch")

install(DIRECTORY xschem_library/examples
                  xschem_library/binto7seg
                  xschem_library/xTAG
                  xschem_library/logic
                  xschem_library/rulz-r8c33
                  xschem_library/pcb
                  xschem_library/ngspice
                  xschem_library/rom8k
                  xschem_library/xschem_simulator
                  xschem_library/symgen
                  xschem_library/gschem_import
        TYPE DOC
        FILES_MATCHING
        PATTERN "*.sym*"
        PATTERN "*.sch"
        PATTERN "stimuli.*")

install(DIRECTORY xschem_library/generators xschem_library/inst_sch_select
        TYPE DOC)

install(DIRECTORY src/utile
        DESTINATION "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}"
        USE_SOURCE_PERMISSIONS
        PATTERN "Makefile" EXCLUDE)

install(DIRECTORY src/systemlib
        DESTINATION "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}")

set(SRC_INSTALL_FILES keys.help xschem.help xschem.tcl break.awk convert_to_verilog2001.awk
flatten.awk flatten_tedax.awk flatten_savenodes.awk make_sym.awk make_sym_lcc.awk
symgen.awk order_labels.awk sort_labels.awk spice.awk tedax.awk verilog.awk
vhdl.awk hspice_backannotate.tcl add_custom_menu.tcl create_graph.tcl
add_custom_button.tcl change_index.tcl icon.xpm resources.tcl xschemrc
ngspice_backannotate.tcl gschemtoxschem.awk traversal.tcl)

list(TRANSFORM SRC_INSTALL_FILES PREPEND "src/")

# Hack to mark awk scripts as executable since install(FILES ...) unfortunately doesn't allow using the source file permissions unlike install(DIRECTORY ...)
set(SRC_INSTALL_FILES_EXECUTABLE ${SRC_INSTALL_FILES})
set(SRC_INSTALL_FILES_NON_EXECUTABLE ${SRC_INSTALL_FILES})
list(FILTER SRC_INSTALL_FILES_EXECUTABLE INCLUDE REGEX ".*\.awk")
list(FILTER SRC_INSTALL_FILES_NON_EXECUTABLE EXCLUDE REGEX ".*\.awk")

install(FILES ${SRC_INSTALL_FILES_NON_EXECUTABLE}
        DESTINATION "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}")
install(PROGRAMS ${SRC_INSTALL_FILES_EXECUTABLE}
        DESTINATION "${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}")

# afaik this is the only way to configure a file with the correct CMAKE_INSTALL_PREFIX when set using `cmake --install . --prefix <prefix>`
string(JOIN "\n" install_script
       "set(install_mandir \"${CMAKE_INSTALL_MANDIR}\")" # CMAKE_INSTALL_MANDIR only available at config time
       [[
         set(/local/xschem/prefix "${CMAKE_INSTALL_PREFIX}") # final CMAKE_INSTALL_PREFIX only known at install time
         configure_file("doc/manpages/xschem.1.in"
                        "${CMAKE_INSTALL_PREFIX}/${install_mandir}/xschem.1"
                        @ONLY)
         file(READ "${CMAKE_INSTALL_PREFIX}/${install_mandir}/xschem.1" file_contents)

         # Strip print[@ @] from file. Could be avoided by simply removing it from the source file since it's not required for cmake
         string(REPLACE "print [@" "" file_contents "${file_contents}")
         string(REPLACE "@]" "" file_contents "${file_contents}")
         file(WRITE "${CMAKE_INSTALL_PREFIX}/${install_mandir}/xschem.1" "${file_contents}")
       ]])

install(CODE "${install_script}")
