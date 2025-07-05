# CMakeList.txt : CMake project for SRAL, include source and define
# project specific logic here.
#
cmake_minimum_required (VERSION 3.25)
set(CMAKE_CXX_STANDARD 20)

# Project options
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_TESTING "Build test executables" ON)
option(BUILD_EXAMPLES "Build example executables" ON)

# Windows-specific options
if(WIN32)
  option(SRAL_USE_STATIC_CRT "Use static C runtime library (/MT)" OFF)
  option(SRAL_ENABLE_UIA "Enable UI Automation support" ON)
endif()

# Configure MSVC runtime library
if(WIN32 AND MSVC)
  if(SRAL_USE_STATIC_CRT)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
  else()
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
  endif()
endif()

set(INCLUDES "Include")

# Enable Hot Reload for MSVC compilers if supported.
if (POLICY CMP0141)
  cmake_policy(SET CMP0141 NEW)
  set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<IF:$<AND:$<C_COMPILER_ID:MSVC>,$<CXX_COMPILER_ID:MSVC>>,$<$<CONFIG:Debug,RelWithDebInfo>:EditAndContinue>,$<$<CONFIG:Debug,RelWithDebInfo>:ProgramDatabase>>")
endif()

project ("SRAL" VERSION 1.0.0 LANGUAGES C CXX)

# Define source files by platform
set(SRAL_COMMON_SOURCES
  "SRC/SRAL.cpp"
  "SRC/Engine.h"
  "SRC/Engine.cpp"
)

set(SRAL_WINDOWS_SOURCES
  "SRC/Encoding.h" "SRC/Encoding.cpp"
  "SRC/NVDA.h" "SRC/NVDA.cpp"
  "SRC/SAPI.h" "SRC/SAPI.cpp"
  "Dep/blastspeak.h" "Dep/blastspeak.c"
  "Dep/fsapi.h" "Dep/fsapi.c"
  "Dep/nvda_control.h" "Dep/nvda_control.c"
  "SRC/Jaws.h" "SRC/Jaws.cpp"
  "Dep/wasapi.h" "Dep/wasapi.cpp"
  "SRC/ZDSR.h" "SRC/ZDSR.cpp"
)

set(SRAL_WINDOWS_UIA_SOURCES
  "SRC/UIA.cpp" "SRC/UIA.h"
  "Dep/UIAProvider.h" "Dep/UIAProvider.cpp"
)

set(SRAL_APPLE_SOURCES
  "SRC/AVSpeech.h" "SRC/AVSpeech.mm"
  "SRC/VoiceOver.h" "SRC/VoiceOver.mm"
)

set(SRAL_LINUX_SOURCES
  "Dep/utf-8.h" "Dep/utf-8.c"
  "SRC/SpeechDispatcher.h" "SRC/SpeechDispatcher.cpp"
)

# Create object library
add_library(${PROJECT_NAME}_obj OBJECT ${SRAL_COMMON_SOURCES})
# Add platform-specific sources
if(WIN32)
  target_sources(${PROJECT_NAME}_obj PRIVATE ${SRAL_WINDOWS_SOURCES})
  if(SRAL_ENABLE_UIA)
    target_sources(${PROJECT_NAME}_obj PRIVATE ${SRAL_WINDOWS_UIA_SOURCES})
  endif()
elseif(APPLE)
  # Enable Objective-C++ for macOS
  enable_language(OBJCXX)
  target_sources(${PROJECT_NAME}_obj PRIVATE ${SRAL_APPLE_SOURCES})
else()
  target_sources(${PROJECT_NAME}_obj PRIVATE ${SRAL_LINUX_SOURCES})
endif()

# Add public headers
target_sources(${PROJECT_NAME}_obj PUBLIC
  FILE_SET HEADERS
  BASE_DIRS "${INCLUDES}"
  FILES "${INCLUDES}/SRAL.h"
)

# Set object library properties
set_property(TARGET ${PROJECT_NAME}_obj PROPERTY POSITION_INDEPENDENT_CODE ON)
target_include_directories(${PROJECT_NAME}_obj PUBLIC ${INCLUDES})

# Configure compile definitions for exports
target_compile_definitions(${PROJECT_NAME}_obj PRIVATE
  $<$<BOOL:${WIN32}>:SRAL_EXPORT>
  $<$<AND:$<BOOL:${WIN32}>,$<BOOL:${SRAL_ENABLE_UIA}>>:SRAL_ENABLE_UIA>
)

# Create libraries based on BUILD_SHARED_LIBS option
if(BUILD_SHARED_LIBS)
  add_library(${PROJECT_NAME} SHARED $<TARGET_OBJECTS:${PROJECT_NAME}_obj>)
  target_include_directories(${PROJECT_NAME} PUBLIC ${INCLUDES})
  set(SRAL_TARGET ${PROJECT_NAME})
else()
  add_library(${PROJECT_NAME} STATIC $<TARGET_OBJECTS:${PROJECT_NAME}_obj>)
  target_include_directories(${PROJECT_NAME} PUBLIC ${INCLUDES})
  target_compile_definitions(${PROJECT_NAME} PUBLIC SRAL_STATIC)
  set(SRAL_TARGET ${PROJECT_NAME})
endif()

# Platform-specific linking
if(WIN32)
  set(WINDOWS_LIBS 
    "user32.lib"
    "kernel32.lib"
    "advapi32.lib"
  )
  
  if(SRAL_ENABLE_UIA)
    list(APPEND WINDOWS_LIBS
      "uiautomationcore.lib" 
      "oleaut32.lib" 
      "ole32.lib"
      "uuid.lib"
      "propsys.lib"
    )
  endif()
  
  target_link_libraries(${SRAL_TARGET} ${WINDOWS_LIBS})
  
  # Add Windows SDK version requirement for UIA functions
  if(MSVC AND SRAL_ENABLE_UIA)
    target_compile_definitions(${SRAL_TARGET} PRIVATE 
      WINVER=0x0A00 
      _WIN32_WINNT=0x0A00
    )
  endif()
elseif(APPLE)
  set(APPLE_FRAMEWORKS
    "-framework AppKit"
    "-framework Foundation"
    "-framework AVFoundation"
  )
  target_link_libraries(${SRAL_TARGET} ${APPLE_FRAMEWORKS})
else()
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(SpeechD REQUIRED speech-dispatcher)

  target_link_libraries(${SRAL_TARGET} ${SpeechD_LIBRARIES})
  target_include_directories(${SRAL_TARGET} PRIVATE ${SpeechD_INCLUDE_DIRS})
  target_compile_options(${SRAL_TARGET} PRIVATE ${SpeechD_CFLAGS_OTHER})

  find_library(BRLAPI_LIB NAMES brlapi HINTS /usr/lib /usr/local/lib)
  if(BRLAPI_LIB)
    message(STATUS "Found BrlAPI: ${BRLAPI_LIB}")
    target_link_libraries(${SRAL_TARGET} ${BRLAPI_LIB})
  else()
    message(WARNING "BrlAPI not found. Braille support will be disabled.")
  endif()
endif()

# Build tests and examples (optional)
if(BUILD_TESTING)
  add_executable(${PROJECT_NAME}_test "Examples/C/SRALExample.c")
  target_link_libraries(${PROJECT_NAME}_test ${SRAL_TARGET})
  target_include_directories(${PROJECT_NAME}_test PRIVATE ${INCLUDES})
endif()

if(BUILD_EXAMPLES AND WIN32)
  add_executable(${PROJECT_NAME}_NVDAControlExConsole
    "Examples/C/NVDAControlExConsole.c"
    "Dep/nvda_control.c"
  )
  target_include_directories(${PROJECT_NAME}_NVDAControlExConsole PRIVATE "Dep")
endif()

# Installation
install(TARGETS ${PROJECT_NAME}_obj FILE_SET HEADERS
  INCLUDES DESTINATION "include"
)

install(TARGETS ${PROJECT_NAME}
  RUNTIME DESTINATION "bin"
  LIBRARY DESTINATION "lib"
  ARCHIVE DESTINATION "lib"
  PUBLIC_HEADER DESTINATION "include"
)

# Display build configuration summary
message(STATUS "")
message(STATUS "═══════════════════════════════════════")
message(STATUS "  SRAL Build Configuration")
message(STATUS "═══════════════════════════════════════")
if(BUILD_SHARED_LIBS)
  message(STATUS "  Library type: Shared (.dll/.so/.dylib)")
else()
  message(STATUS "  Library type: Static (.lib/.a)")
endif()
message(STATUS "  Target name: ${PROJECT_NAME}")
message(STATUS "  Platform: ${CMAKE_SYSTEM_NAME}")
if(WIN32)
  if(MSVC)
    if(SRAL_USE_STATIC_CRT)
      message(STATUS "  Windows CRT: Static (/MT)")
    else()
      message(STATUS "  Windows CRT: Dynamic (/MD)")
    endif()
  endif()
  message(STATUS "  Static CRT option: ${SRAL_USE_STATIC_CRT}")
  message(STATUS "  UIA support: ${SRAL_ENABLE_UIA}")
endif()
if(CMAKE_BUILD_TYPE)
  message(STATUS "  Build type: ${CMAKE_BUILD_TYPE}")
endif()
message(STATUS "  Testing: ${BUILD_TESTING}")
message(STATUS "  Examples: ${BUILD_EXAMPLES}")
message(STATUS "═══════════════════════════════════════")
message(STATUS "")
