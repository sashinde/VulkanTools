project(vktraceviewer)
cmake_minimum_required(VERSION 2.8.9)

include("${SRC_DIR}/build_options.cmake")

set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} "${SRC_DIR}/cmake/Modules/")

# we want cmake to link the Qt libs into the binary
# This policy was introduced in 2.8.11, so on newer versions, use the OLD policy to maintain consistent behavior
if (POLICY CMP0020)
   cmake_policy(SET CMP0020 OLD)
endif()

find_package(Qt5 COMPONENTS Widgets Gui Core Svg QUIET)

if(NOT Qt5_FOUND)
# After Qt5.6 is installed, you may need to add the following to the cmake command line:
# -DCMAKE_PREFIX_PATH=C:\\Qt\\5.6\\msvc2015_64\\
message(WARNING "WARNING: vktraceviewer will be excluded because Qt5 was not found.")
else()

find_package(Threads REQUIRED)
find_package(X11 REQUIRED)

require_pthreads()

include_directories(
    ${SRC_DIR}
    ${SRC_DIR}/vktrace_common
    ${SRC_DIR}/vktrace_replay
    ${SRC_DIR}/vktrace_extensions/vktracevulkan/vkreplay
    ${CMAKE_CURRENT_BINARY_DIR}
    ${Qt5Widgets_INCLUDE_DIRS}
)

set(SRC_LIST
    main.cpp
    vktraceviewer.cpp
    vktraceviewer_settings.cpp
    vktraceviewer_output.cpp
    vktraceviewer_trace_file_utils.cpp
    vktraceviewer_qgeneratetracedialog.cpp
    vktraceviewer_qsettingsdialog.cpp
    vktraceviewer_qtimelineview.cpp
    vktraceviewer_qtracefileloader.cpp
    vktraceviewer_QReplayWorker.cpp
    vktraceviewer_controller_factory.cpp
    ${SRC_DIR}/vktrace_replay/vkreplay_seq.cpp
    ${SRC_DIR}/vktrace_replay/vkreplay_factory.cpp
   )

# This should only contain headers that define a QOBJECT
# Typically that means just headers for UI objects
set(UI_HEADER_LIST
    vktraceviewer.h
    vktraceviewer_qgeneratetracedialog.h
    vktraceviewer_qsettingsdialog.h
    vktraceviewer_qtimelineview.h
    vktraceviewer_qimageviewer.h
    vktraceviewer_qsvgviewer.h
    vktraceviewer_QReplayWidget.h
    vktraceviewer_QReplayWorker.h
    vktraceviewer_QTraceFileModel.h
    vktraceviewer_qtracefileloader.h
   )

# These is for all headers
set(HEADER_LIST
    vktraceviewer.h
    vktraceviewer_settings.h
    vktraceviewer_output.h
    vktraceviewer_controller.h
    vktraceviewer_controller_factory.h
    vktraceviewer_qgeneratetracedialog.h
    vktraceviewer_qsettingsdialog.h
    vktraceviewer_qtimelineview.h
    vktraceviewer_qimageviewer.h
    vktraceviewer_qsvgviewer.h
    vktraceviewer_QReplayWidget.h
    vktraceviewer_QReplayWorker.h
    vktraceviewer_qtracefileloader.h
    vktraceviewer_QTraceFileModel.h
    vktraceviewer_trace_file_utils.h
    vktraceviewer_view.h
   )

set(FORM_LIST
    vktraceviewer.ui
   )

set(RESOURCE_LIST
   )

QT5_WRAP_CPP(QT_GEN_HEADER_MOC_LIST ${UI_HEADER_LIST})
QT5_WRAP_UI(QT_GEN_FORM_HEADER_LIST ${FORM_LIST})
QT5_ADD_RESOURCES(QT_GEN_RESOURCE_RCC_LIST ${RESOURCE_LIST})

if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang")
    add_compiler_flag("-Wno-global-constructors -Wno-used-but-marked-unused")
endif()

# Platform specific compile flags.
if (NOT MSVC)
    add_compiler_flag("-fPIC")
endif()

add_executable(${PROJECT_NAME} ${SRC_LIST} ${HEADER_LIST}
    ${QT_GEN_HEADER_MOC_LIST}
    ${QT_GEN_FORM_HEADER_LIST}
    ${QT_GEN_RESOURCE_RCC_LIST}
   )

if (TARGET SDL)
    add_dependencies(${PROJECT_NAME} SDL)
endif ()

add_dependencies(${PROJECT_NAME} vktraceviewer_vk)

target_link_libraries(${PROJECT_NAME}
    Qt5::Widgets
    Qt5::Core
    Qt5::Svg
    vktrace_common
    vulkan_replay
    ${CMAKE_DL_LIBS}
    ${X11_X11_LIB}
)

if (MSVC)
  # copy the debug and release dlls for Qt5Widgets, Qt5Core, and Qt5Gui
  set (COPY_DEST ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${CMAKE_CFG_INTDIR})

  get_property(qt5_qmake_executable TARGET ${Qt5Core_QMAKE_EXECUTABLE} PROPERTY IMPORT_LOCATION)
  execute_process(COMMAND ${qt5_qmake_executable} -query QT_INSTALL_PLUGINS OUTPUT_VARIABLE QT_INSTALL_PLUGINS OUTPUT_STRIP_TRAILING_WHITESPACE)
  execute_process(COMMAND ${qt5_qmake_executable} -query QT_INSTALL_BINS OUTPUT_VARIABLE QT_INSTALL_BINS OUTPUT_STRIP_TRAILING_WHITESPACE)

  # There are also several other files that need to be copied or created
  FILE(WRITE ${CMAKE_CURRENT_LIST_DIR}/qt.conf "[Paths]\nPlugins=.")
  add_custom_target(copy_deps_qt5 ALL
                    ${CMAKE_COMMAND} -E make_directory "${COPY_DEST}/platforms"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_CURRENT_LIST_DIR}/qt.conf" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Widgets.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Widgetsd.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Core.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Cored.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Gui.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Guid.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Svgd.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/Qt5Svg.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/icudt52.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/icuin52.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/icuuc52.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/libGLESv2d.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/libGLESv2.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/libEGLd.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_BINS}/libEGL.dll" "${COPY_DEST}"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_PLUGINS}/platforms/qwindows.dll" "${COPY_DEST}/platforms"
                    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${QT_INSTALL_PLUGINS}/platforms/qwindowsd.dll"  "${COPY_DEST}/platforms"
                    COMMENT "Copying vktraceviewer Qt5 dependencies to ${COPY_DEST}"
                    VERBATIM)

endif()

build_options_finalize()
endif(NOT Qt5_FOUND)
