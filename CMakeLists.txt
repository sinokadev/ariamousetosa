cmake_minimum_required(VERSION 3.16)
project(Aria LANGUAGES C CXX)

include(FetchContent)

# ---------------- Renderer option ----------------
set(RENDERER "wxwidgets" CACHE STRING "Renderer: wxwidgets or opengl")
set_property(CACHE RENDERER PROPERTY STRINGS wxwidgets opengl)

if(RENDERER STREQUAL "opengl")
  add_compile_definitions(RENDERER_OPENGL)
elseif(RENDERER STREQUAL "wxwidgets")
  add_compile_definitions(RENDERER_WXWIDGETS)
else()
  message(FATAL_ERROR "Unknown renderer: ${RENDERER}")
endif()

# ---------------- Sources (ONLY your app) ----------------
file(GLOB_RECURSE SOURCES CONFIGURE_DEPENDS
  "${CMAKE_SOURCE_DIR}/src/*.cpp"
)
add_executable(Aria ${SOURCES})
target_compile_features(Aria PRIVATE cxx_std_11)

target_include_directories(Aria PRIVATE
  "${CMAKE_SOURCE_DIR}/src"
  "${CMAKE_SOURCE_DIR}"
)

# ---------------- Build flags ----------------
target_compile_options(Aria PRIVATE
  $<$<CONFIG:Debug>:-g -Wall -Wextra -Wno-unused-parameter -Wfatal-errors>
  $<$<CONFIG:Release>:$<$<PLATFORM_ID:Darwin>:-O1> $<$<NOT:$<PLATFORM_ID:Darwin>>:-O2>>
)
target_compile_definitions(Aria PRIVATE
  $<$<CONFIG:Debug>:_MORE_DEBUG_CHECKS _CHECK_FOR_LEAKS DEBUG=1>
  $<$<CONFIG:Release>:NDEBUG=1>
)

# ---------------- wxWidgets ----------------
find_package(wxWidgets REQUIRED COMPONENTS core base adv net webview)
include(${wxWidgets_USE_FILE})
target_link_libraries(Aria PRIVATE ${wxWidgets_LIBRARIES})

# =============================================================================
# Dependencies: RtMidi / JDKSMIDI / irrXML
# =============================================================================

# ---- RtMidi ----
option(USE_SYSTEM_RTMIDI "Use system-installed RtMidi (if available)" ON)

if(USE_SYSTEM_RTMIDI)
  find_path(RTMIDI_INCLUDE_DIR RtMidi.h)
  find_library(RTMIDI_LIBRARY rtmidi)
endif()

if(RTMIDI_INCLUDE_DIR AND RTMIDI_LIBRARY)
  message(STATUS "Using system RtMidi: ${RTMIDI_LIBRARY}")
  target_include_directories(Aria PRIVATE ${RTMIDI_INCLUDE_DIR})
  target_link_libraries(Aria PRIVATE ${RTMIDI_LIBRARY})
else()
  message(STATUS "Fetching RtMidi via FetchContent")
  FetchContent_Declare(
    rtmidi
    GIT_REPOSITORY https://github.com/thestk/rtmidi.git
    GIT_TAG        master
  )
  FetchContent_MakeAvailable(rtmidi)

  if(NOT TARGET rtmidi)
    add_library(rtmidi STATIC
      "${rtmidi_SOURCE_DIR}/RtMidi.cpp"
    )
    target_include_directories(rtmidi PUBLIC "${rtmidi_SOURCE_DIR}")
  endif()

  target_link_libraries(Aria PRIVATE rtmidi)
endif()

# ---- JDKSMIDI ----
option(USE_SYSTEM_JDKSMIDI "Use system-installed jdksmidi (rare)" OFF)

if(USE_SYSTEM_JDKSMIDI)
  find_path(JDKSMIDI_INCLUDE_DIR jdksmidi/world.h)
  find_library(JDKSMIDI_LIBRARY jdksmidi)
endif()

if(JDKSMIDI_INCLUDE_DIR AND JDKSMIDI_LIBRARY)
  message(STATUS "Using system jdksmidi: ${JDKSMIDI_LIBRARY}")
  target_include_directories(Aria PRIVATE ${JDKSMIDI_INCLUDE_DIR})
  target_link_libraries(Aria PRIVATE ${JDKSMIDI_LIBRARY})
else()
  message(STATUS "Fetching jdksmidi via FetchContent")
  FetchContent_Declare(
    jdksmidi
    GIT_REPOSITORY https://github.com/jdkoftinoff/jdksmidi.git
    GIT_TAG        master
  )
  FetchContent_Populate(jdksmidi)

  file(GLOB_RECURSE JDKSMIDI_SOURCES CONFIGURE_DEPENDS
    "${jdksmidi_SOURCE_DIR}/src/*.cpp"
  )
  add_library(jdksmidi STATIC ${JDKSMIDI_SOURCES})
  target_include_directories(jdksmidi PUBLIC
    "${jdksmidi_SOURCE_DIR}/include"
    "${jdksmidi_SOURCE_DIR}/src"
  )

  target_link_libraries(Aria PRIVATE jdksmidi)
endif()

# ---- irrXML ----
option(USE_SYSTEM_IRRXML "Use system-installed irrXML" ON)

if(USE_SYSTEM_IRRXML)
  find_library(IRRXML_LIB irrxml IrrXML)
  find_path(IRRXML_INCLUDE_DIR irrXML.h)
endif()

if(IRRXML_LIB AND IRRXML_INCLUDE_DIR)
  message(STATUS "Using system irrXML: ${IRRXML_LIB}")
  target_include_directories(Aria PRIVATE ${IRRXML_INCLUDE_DIR})
  target_link_libraries(Aria PRIVATE ${IRRXML_LIB})
else()
  message(STATUS "Fetching irrXML (zip) via FetchContent")

  # CMP0135 경고 정리
  if(POLICY CMP0135)
    cmake_policy(SET CMP0135 NEW)
  endif()

  FetchContent_Declare(
    irrxml
    URL https://prdownloads.sourceforge.net/irrlicht/irrxml-1.2.zip
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
  )
  FetchContent_Populate(irrxml)

  # irrXML 소스 / 헤더 탐색
  file(GLOB_RECURSE IRRXML_CPP CONFIGURE_DEPENDS
    "${irrxml_SOURCE_DIR}/*irrXML.cpp"
  )
  file(GLOB_RECURSE IRRXML_HDR CONFIGURE_DEPENDS
    "${irrxml_SOURCE_DIR}/*irrXML.h"
    "${irrxml_SOURCE_DIR}/*irrTypes.h"
    "${irrxml_SOURCE_DIR}/*fast_atof.h"
  )

  if(NOT IRRXML_CPP OR NOT IRRXML_HDR)
    message(FATAL_ERROR
      "Fetched irrXML but required irrXML sources not found under: ${irrxml_SOURCE_DIR}"
    )
  endif()

  # static library 생성
  add_library(irrxml STATIC ${IRRXML_CPP})

  # include 구조 맞추기: irrXML/irrXML.h
  set(IRRXML_INCLUDE_ROOT "${CMAKE_BINARY_DIR}/irrxml_include")
  file(MAKE_DIRECTORY "${IRRXML_INCLUDE_ROOT}/irrXML")

  foreach(_hdr ${IRRXML_HDR})
    file(COPY "${_hdr}"
         DESTINATION "${IRRXML_INCLUDE_ROOT}/irrXML")
  endforeach()

  target_include_directories(irrxml PUBLIC
    "${IRRXML_INCLUDE_ROOT}"
  )

  # Aria에 연결
  target_link_libraries(Aria PRIVATE irrxml)
  target_include_directories(Aria
    PRIVATE
      ${CMAKE_SOURCE_DIR}/src
    PUBLIC
      ${CMAKE_SOURCE_DIR}/include
  )

endif()


# =============================================================================
# Platform specifics
# =============================================================================
option(USE_JACK "Enable JACK MIDI" OFF)

if(APPLE)
  add_compile_definitions(_MAC_QUICKTIME_COREAUDIO)
  enable_language(OBJCXX)

  target_sources(Aria PRIVATE
    "${CMAKE_SOURCE_DIR}/src/Midi/Players/Mac/QuickTimeExport.mm"
    "${CMAKE_SOURCE_DIR}/src/GUI/Machelper.mm"
  )
  target_include_directories(Aria PRIVATE "${CMAKE_SOURCE_DIR}/src/Midi/Players/Mac")

  target_link_libraries(Aria PRIVATE
    "-framework CoreAudio"
    "-framework AudioToolbox"
    "-framework AudioUnit"
    "-framework AppKit"
    "-framework Carbon"
    "-framework Cocoa"
    "-framework IOKit"
    "-framework CoreMIDI"
  )

  if(RENDERER STREQUAL "opengl")
    target_link_libraries(Aria PRIVATE "-framework OpenGL")
  endif()

elseif(UNIX)
  add_compile_definitions(_ALSA)

  find_package(PkgConfig REQUIRED)
  pkg_check_modules(GLIB2 REQUIRED glib-2.0)
  target_include_directories(Aria PRIVATE ${GLIB2_INCLUDE_DIRS})
  target_link_libraries(Aria PRIVATE ${GLIB2_LIBRARIES})

  if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    pkg_check_modules(ALSA REQUIRED alsa)
    target_include_directories(Aria PRIVATE ${ALSA_INCLUDE_DIRS})
    target_link_libraries(Aria PRIVATE ${ALSA_LIBRARIES} dl m)
  endif()

  if(RENDERER STREQUAL "opengl")
    add_compile_definitions(wxUSE_GLCANVAS=1)
    find_package(OpenGL REQUIRED)
    target_link_libraries(Aria PRIVATE OpenGL::GL OpenGL::GLU)
  endif()
endif()

if(USE_JACK)
  add_compile_definitions(USE_JACK)
  find_library(JACK_LIB jack)
  if(JACK_LIB)
    target_link_libraries(Aria PRIVATE ${JACK_LIB})
  else()
    message(WARNING "USE_JACK is ON but libjack not found")
  endif()
endif()

# ---- install ----
include(GNUInstallDirs)
install(TARGETS Aria RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

install(DIRECTORY "${CMAKE_SOURCE_DIR}/Resources/"
  DESTINATION "${CMAKE_INSTALL_DATADIR}/Aria"
  PATTERN ".svn" EXCLUDE
  PATTERN "*.icns" EXCLUDE
)

file(GLOB MO_FILES RELATIVE "${CMAKE_SOURCE_DIR}" "international/*/aria_maestosa.mo")
foreach(MO ${MO_FILES})
  string(REPLACE "international/" "" TMP "${MO}")
  string(REPLACE "/aria_maestosa.mo" "" LANG "${TMP}")
  install(FILES "${CMAKE_SOURCE_DIR}/${MO}"
    DESTINATION "${CMAKE_INSTALL_DATADIR}/locale/${LANG}/LC_MESSAGES"
    RENAME "aria_maestosa.mo"
  )
endforeach()
