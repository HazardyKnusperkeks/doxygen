if (force_qt STREQUAL "Qt5" OR NOT force_qt)
  find_package(Qt5Core QUIET CONFIG)
  if (Qt5Core_FOUND)
    message(STATUS "Using Qt5")
    find_package(Qt5 COMPONENTS Widgets Gui Xml)
    macro(qt_wrap_cpp)
      qt5_wrap_cpp(${ARGN})
    endmacro()
    macro(qt_add_resources)
      qt5_add_resources(${ARGN})
    endmacro()
  else()
    if (NOT force_qt)
      find_package(Qt6Core QUIET CONFIG)
      if (Qt6Core_FOUND)
        message(STATUS "Using Qt6")
        find_package(Qt6 COMPONENTS Widgets Gui Xml)
        macro(qt_wrap_cpp)
          qt6_wrap_cpp(${ARGN})
        endmacro()
        macro(qt_add_resources)
          qt6_add_resources(${ARGN})
        endmacro()
      else()
        message(FATAL_ERROR "Qt5 nor Qt6 found")
      endif()
    else()
      message(FATAL_ERROR "Qt5 not found")
    endif()
  endif()
else()
  find_package(Qt6Core QUIET CONFIG)
  if (Qt6Core_FOUND)
    message(STATUS "Using Qt6")
    find_package(Qt6 COMPONENTS Widgets Gui Xml)
    macro(qt_wrap_cpp)
      qt6_wrap_cpp(${ARGN})
    endmacro()
    macro(qt_add_resources)
      qt6_add_resources(${ARGN})
    endmacro()
  else()
    message(FATAL_ERROR "Qt6 not found")
  endif()
endif()

include_directories(
  .
  ${PROJECT_SOURCE_DIR}/libversion
  ${GENERATED_SRC}
)

set(GENERATED_SRC_WIZARD ${GENERATED_SRC}/doxywizard)
file(MAKE_DIRECTORY ${GENERATED_SRC_WIZARD})

add_definitions(-DQT_NO_CAST_FROM_ASCII -DQT_NO_CAST_TO_ASCII -DUNICODE)
if (NOT Qt6Core_FOUND AND NOT Qt5Core_FOUND)
  include(${QT_USE_FILE})
endif()

# generate settings.h
file(GENERATE OUTPUT ${GENERATED_SRC_WIZARD}/settings.h
CONTENT "#ifndef SETTINGS_H
#define SETTINGS_H
#define USE_SQLITE3 ${sqlite3}
#define USE_LIBCLANG ${clang}
#define IS_SUPPORTED(x) \\
  ((USE_SQLITE3  && strcmp(\"USE_SQLITE3\",(x))==0)  || \\
   (USE_LIBCLANG && strcmp(\"USE_LIBCLANG\",(x))==0) || \\
  0)
#endif" )
set_source_files_properties(${GENERATED_SRC_WIZARD}/settings.h PROPERTIES GENERATED 1)

# generate configdoc.cpp
add_custom_command(
    COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/src/configgen.py -wiz ${PROJECT_SOURCE_DIR}/src/config.xml > ${GENERATED_SRC_WIZARD}/configdoc.cpp
    DEPENDS  ${PROJECT_SOURCE_DIR}/src/configgen.py ${PROJECT_SOURCE_DIR}/src/config.xml
    OUTPUT ${GENERATED_SRC_WIZARD}/configdoc.cpp
)
set_source_files_properties(${GENERATED_SRC_WIZARD}/configdoc.cpp PROPERTIES GENERATED 1)

set(LEX_FILES config_doxyw)
foreach(lex_file ${LEX_FILES})
    add_custom_command(
        COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/src/scan_states.py ${PROJECT_SOURCE_DIR}/addon/doxywizard/${lex_file}.l > ${GENERATED_SRC_WIZARD}/${lex_file}.l.h
        DEPENDS ${PROJECT_SOURCE_DIR}/src/scan_states.py ${PROJECT_SOURCE_DIR}/addon/doxywizard/${lex_file}.l
        OUTPUT  ${GENERATED_SRC_WIZARD}/${lex_file}.l.h
    )
    set_source_files_properties(${GENERATED_SRC_WIZARD}/${lex_file}.l.h PROPERTIES GENERATED 1)

    FLEX_TARGET(${lex_file}        ${lex_file}.l        ${GENERATED_SRC_WIZARD}/${lex_file}.cpp        COMPILE_FLAGS "${LEX_FLAGS}")
endforeach()

qt_wrap_cpp(doxywizard_MOC
doxywizard.h
expert.h
helplabel.h
inputint.h
inputbool.h
inputstring.h
inputstrlist.h
wizard.h
)

qt_add_resources(doxywizard_RESOURCES_RCC doxywizard.qrc)

add_executable(doxywizard WIN32
config_msg.cpp
doxywizard.cpp
expert.cpp
wizard.cpp
inputbool.cpp
inputstring.cpp
inputint.cpp
inputstrlist.cpp
${GENERATED_SRC_WIZARD}/settings.h
${GENERATED_SRC_WIZARD}/config_doxyw.cpp
${GENERATED_SRC_WIZARD}/config_doxyw.l.h
${GENERATED_SRC_WIZARD}/configdoc.cpp
${doxywizard_MOC}
${doxywizard_RESOURCES_RCC}
doxywizard.rc
)

set_property(TARGET doxywizard PROPERTY WIN32_EXECUTABLE true)

if(Qt5Core_FOUND)
  target_link_libraries(doxywizard Qt5::Core Qt5::Gui Qt5::Widgets Qt5::Xml doxygen_version)
else()
if(Qt6Core_FOUND)
  target_link_libraries(doxywizard Qt6::Core Qt6::Gui Qt6::Widgets Qt6::Xml doxygen_version)
else()
  target_link_libraries(doxywizard ${QT_LIBRARIES} ${QT_QTMAIN_LIBRARY} doxygen_version)
endif()
endif()

install(TARGETS doxywizard DESTINATION bin)
