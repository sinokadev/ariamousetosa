cmake_minimum_required(VERSION 3.16)
project(Aria LANGUAGES C CXX)

set(RENDERER "wxwidgets" CACHE STRING "Renderer: wxwidgets or opengl")
set_property(CACHE RENDERER PROPERTY STRINGS wxwidgets opengl)

if(RENDERER STREQUAL "opengl")
  add_compile_definitions(RENDERER_OPENGL)
elseif(RENDERER STREQUAL "wxwidgets")
  add_compile_definitions(RENDERER_WXWIDGETS)
else()
  message(FATAL_ERROR "Unknown renderer: ${RENDERER}")
endif()

# ---- sources + target ----
file(GLOB_RECURSE SOURCES CONFIGURE_DEPENDS
  "${CMAKE_SOURCE_DIR}/Src/*.cpp"
  "${CMAKE_SOURCE_DIR}/rtmidi/*.cpp"
  "${CMAKE_SOURCE_DIR}/libjdkmidi/*.cpp"
)
add_executable(Aria ${SOURCES})
target_compile_features(Aria PRIVATE cxx_std_11)

# include paths
target_include_directories(Aria PRIVATE
  "${CMAKE_SOURCE_DIR}/Src" "${CMAKE_SOURCE_DIR}"
  "${CMAKE_SOURCE_DIR}/libjdkmidi/include" "${CMAKE_SOURCE_DIR}/rtmidi"
)

# build type flags (Debug/Release)
target_compile_options(Aria PRIVATE
  $<$<CONFIG:Debug>:-g -Wall -Wextra -Wno-unused-parameter -Wfatal-errors>
  $<$<CONFIG:Release>:$<$<PLATFORM_ID:Darwin>:-O1> $<$<NOT:$<PLATFORM_ID:Darwin>>:-O2>>
)
target_compile_definitions(Aria PRIVATE
  $<$<CONFIG:Debug>:_MORE_DEBUG_CHECKS _CHECK_FOR_LEAKS DEBUG=1>
  $<$<CONFIG:Release>:NDEBUG=1>
)

# ---- wxWidgets ----
find_package(wxWidgets REQUIRED COMPONENTS core base adv net webview)
include(${wxWidgets_USE_FILE})
target_link_libraries(Aria PRIVATE ${wxWidgets_LIBRARIES})

# ---- IrrXML / Irrlicht (AriaFileWriter에서 사용) ----
find_library(IRRLICHT_LIB Irrlicht)
if(IRRLICHT_LIB)
  target_link_libraries(Aria PRIVATE ${IRRLICHT_LIB})
else()
  find_library(IRRXML_LIB IrrXML irrxml)
  if(IRRXML_LIB)
    target_link_libraries(Aria PRIVATE ${IRRXML_LIB})
  else()
    message(FATAL_ERROR "IrrXML/Irrlicht not found (needed for irr::io::createIrrXMLReader)")
  endif()
endif()

option(USE_JACK "Enable JACK MIDI" OFF)

# ---- platform specific ----
if(APPLE)
  add_compile_definitions(_MAC_QUICKTIME_COREAUDIO)
  enable_language(OBJCXX)

  target_sources(Aria PRIVATE
    "${CMAKE_SOURCE_DIR}/Src/Midi/Players/Mac/QuickTimeExport.mm"
    "${CMAKE_SOURCE_DIR}/Src/GUI/Machelper.mm"
  )
  target_include_directories(Aria PRIVATE "${CMAKE_SOURCE_DIR}/Src/Midi/Players/Mac")

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

# JACK (요청 시)
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

install(TARGETS Aria
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

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
