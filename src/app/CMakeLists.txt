# Generate and configure translation files
# -----------------------------------------------------------------------------
# -----------------------------------------------------------------------------
# Based on https://gist.github.com/giraldeau/546ba5512a74dfe9d8ea0862d66db412
file(GLOB QBT_TS_FILES "${qBittorrent_SOURCE_DIR}/src/lang/*.ts")
set_source_files_properties(${QBT_TS_FILES} PROPERTIES OUTPUT_LOCATION "${qBittorrent_BINARY_DIR}/src/lang")
qt_add_translation(QBT_QM_FILES ${QBT_TS_FILES} OPTIONS -silent)
configure_file("${qBittorrent_SOURCE_DIR}/src/lang/lang.qrc" "${qBittorrent_BINARY_DIR}/src/lang/lang.qrc" COPYONLY)

if (WEBUI)
    file(GLOB QBT_WEBUI_TS_FILES "${qBittorrent_SOURCE_DIR}/src/webui/www/translations/*.ts")
    set_source_files_properties(${QBT_WEBUI_TS_FILES}
        PROPERTIES OUTPUT_LOCATION "${qBittorrent_BINARY_DIR}/src/webui/www/translations")
    qt_add_translation(QBT_WEBUI_QM_FILES ${QBT_WEBUI_TS_FILES} OPTIONS -silent)
    configure_file("${qBittorrent_SOURCE_DIR}/src/webui/www/translations/webui_translations.qrc"
        "${qBittorrent_BINARY_DIR}/src/webui/www/translations/webui_translations.qrc" COPYONLY)
endif()

FILE(GLOB QT_TRANSLATIONS "${qBittorrent_SOURCE_DIR}/dist/qt-translations/qtbase_*.qm")
foreach(EXTRA_TRANSLATION IN ITEMS "fa" "gl" "lt" "pt" "sl" "sv" "zh_CN")
    list(APPEND QT_TRANSLATIONS "${qBittorrent_SOURCE_DIR}/dist/qt-translations/qt_${EXTRA_TRANSLATION}.qm")
endforeach()

# Executable target configuration
# -----------------------------------------------------------------------------
# -----------------------------------------------------------------------------
add_executable(qbt_app)

target_sources(qbt_app PRIVATE
    # headers
    application.h
    applicationinstancemanager.h
    cmdoptions.h
    filelogger.h
    qtlocalpeer/qtlocalpeer.h
    upgrade.h

    # sources
    application.cpp
    applicationinstancemanager.cpp
    cmdoptions.cpp
    filelogger.cpp
    main.cpp
    qtlocalpeer/qtlocalpeer.cpp
    upgrade.cpp

    # resources
    "${qBittorrent_SOURCE_DIR}/src/icons/icons.qrc"
    "${qBittorrent_SOURCE_DIR}/src/searchengine/searchengine.qrc"
    ${QBT_QM_FILES}
    "${qBittorrent_BINARY_DIR}/src/lang/lang.qrc" # yes, it's supposed to be "*_BINARY_DIR"
)

target_link_libraries(qbt_app PRIVATE
    qbt_base
)

set_target_properties(qbt_app PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}")
if (GUI)
    set_target_properties(qbt_app PROPERTIES OUTPUT_NAME qbittorrent)
else()
    set_target_properties(qbt_app PROPERTIES OUTPUT_NAME qbittorrent-nox)
endif()

# Additional platform specific configuration
# -----------------------------------------------------------------------------
# -----------------------------------------------------------------------------
set_source_files_properties(${QT_TRANSLATIONS} PROPERTIES MACOSX_PACKAGE_LOCATION translations)
set_source_files_properties(
    "${qBittorrent_SOURCE_DIR}/dist/mac/qt.conf"
    "${qBittorrent_SOURCE_DIR}/dist/mac/qBitTorrentDocument.icns"
    "${qBittorrent_SOURCE_DIR}/dist/mac/qbittorrent_mac.icns"
        PROPERTIES
            MACOSX_PACKAGE_LOCATION Resources
)

if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    # provide variables for substitution in dist/mac/Info.plist
    get_target_property(EXECUTABLE_NAME qbt_app OUTPUT_NAME)
    # This variable name should be changed once qmake is no longer used. Refer to the discussion in PR #14813
    set(MACOSX_DEPLOYMENT_TARGET ${CMAKE_OSX_DEPLOYMENT_TARGET})
    set_target_properties(qbt_app PROPERTIES
        MACOSX_BUNDLE ON
        MACOSX_BUNDLE_BUNDLE_NAME "qBittorrent"
        MACOSX_BUNDLE_INFO_PLIST ${qBittorrent_SOURCE_DIR}/dist/mac/Info.plist
    )
    target_sources(qbt_app PRIVATE
        ${QT_TRANSLATIONS}
        ${qBittorrent_SOURCE_DIR}/dist/mac/qt.conf
        ${qBittorrent_SOURCE_DIR}/dist/mac/qBitTorrentDocument.icns
        ${qBittorrent_SOURCE_DIR}/dist/mac/qbittorrent_mac.icns
    )
elseif (CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set_target_properties(qbt_app PROPERTIES WIN32_EXECUTABLE ON)
    if (MINGW)
        target_sources(qbt_app PRIVATE ${qBittorrent_SOURCE_DIR}/src/qbittorrent_mingw.rc)
    else()
        target_sources(qbt_app PRIVATE ${qBittorrent_SOURCE_DIR}/src/qbittorrent.rc)
    endif()
    target_sources(qbt_app PRIVATE ${qBittorrent_SOURCE_DIR}/src/qbittorrent.exe.manifest)
endif()

# Additional feature dependent configuration
# -----------------------------------------------------------------------------
# -----------------------------------------------------------------------------
if (STACKTRACE)
    target_compile_definitions(qbt_app PRIVATE STACKTRACE)

    if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
        target_sources(qbt_app PRIVATE stacktrace_win.h)

        if (GUI)
            qt_wrap_ui(STACKTRACE_UI_HEADERS stacktracedialog.ui)

            target_sources(qbt_app PRIVATE
                stacktracedialog.h
                stacktracedialog.cpp
                ${STACKTRACE_UI_HEADERS}
            )

            # UI headers will be generated in ${CMAKE_CURRENT_BINARY_DIR}
            target_include_directories(qbt_app PRIVATE
                ${CMAKE_CURRENT_BINARY_DIR}
            )
        endif()

        # i686 arch on Windows requires frame pointer preservation
        if (MSVC)
            target_compile_options(qbt_app PRIVATE /Zi)
            target_link_options(qbt_app PUBLIC LINKER:/DEBUG)
            if (CMAKE_SIZEOF_VOID_P EQUAL 4)
                target_compile_options(qbt_app PRIVATE /Oy)
            endif()
        else()
            if (CMAKE_SIZEOF_VOID_P EQUAL 4)
                target_compile_options(qbt_app PRIVATE -fno-omit-frame-pointer)
            endif()
        endif()

        target_link_libraries(qbt_app PUBLIC dbghelp)
    else()
        target_sources(qbt_app PRIVATE stacktrace.h)
    endif()
endif()

if (GUI)
    target_link_libraries(qbt_app PRIVATE qbt_gui)
    if ((CMAKE_SYSTEM_NAME STREQUAL "Windows") OR (CMAKE_SYSTEM_NAME STREQUAL "Darwin"))
        qt_import_plugins(qbt_app INCLUDE Qt::QSvgIconPlugin Qt::QSvgPlugin)
    endif()
endif()

if (WEBUI)
    target_sources(qbt_app PRIVATE
        ${QBT_WEBUI_QM_FILES}
        ${qBittorrent_BINARY_DIR}/src/webui/www/translations/webui_translations.qrc # yes, it's supposed to be "*_BINARY_DIR"
    )
    target_link_libraries(qbt_app PRIVATE qbt_webui)
endif()

# Installation
# -----------------------------------------------------------------------------
# -----------------------------------------------------------------------------
install(TARGETS qbt_app
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    BUNDLE  DESTINATION .
    COMPONENT runtime
)

if (MSVC)
    install(FILES $<TARGET_PDB_FILE:qbt_app>
        DESTINATION ${CMAKE_INSTALL_BINDIR}
        OPTIONAL
    )
endif()
