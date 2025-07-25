# Copyright (c) Microsoft Corporation.
# Licensed under the MIT License.

cmake_minimum_required (VERSION 3.20)

set(DIRECTXTEX_VERSION 2.0.8)

if(XBOX_CONSOLE_TARGET STREQUAL "durango")
  set(CMAKE_TRY_COMPILE_TARGET_TYPE "STATIC_LIBRARY")
endif()

project (DirectXTex
  VERSION ${DIRECTXTEX_VERSION}
  DESCRIPTION "DirectX Texture Library"
  HOMEPAGE_URL "https://go.microsoft.com/fwlink/?LinkId=248926"
  LANGUAGES CXX)

if(DEFINED XBOX_CONSOLE_TARGET)
  set(CMAKE_CXX_STANDARD_LIBRARIES "")
endif()

option(BUILD_TOOLS "Build tex command-line tools" ON)

option(BUILD_SHARED_LIBS "Build DirectXTex as a shared library" OFF)

option(BUILD_SAMPLE "Build DDSView sample (requires fxc.exe)" ON)

# Includes the functions for Direct3D 11 resources and DirectCompute compression
option(BUILD_DX11 "Build with DirectX11 Runtime support (requires fxc.exe)" ON)

# Includes the functions for creating Direct3D 12 resources at runtime
option(BUILD_DX12 "Build with DirectX12 Runtime support" ON)

# Enable the use of OpenMP for software BC6H/BC7 compression
option(BC_USE_OPENMP "Build with OpenMP support" ON)

# Builds Xbox extensions for Host PC
option(BUILD_XBOX_EXTS_XBOXONE "Build Xbox library extensions for Xbox One" OFF)
option(BUILD_XBOX_EXTS_SCARLETT "Build Xbox library extensions for Xbox Series X|S" OFF)

# https://devblogs.microsoft.com/cppblog/spectre-mitigations-in-msvc/
option(ENABLE_SPECTRE_MITIGATION "Build using /Qspectre for MSVC" OFF)

option(DISABLE_MSVC_ITERATOR_DEBUGGING "Disable iterator debugging in Debug configurations with the MSVC CRT" OFF)

option(ENABLE_CODE_ANALYSIS "Use Static Code Analysis on build" OFF)

option(ENABLE_CODE_COVERAGE "Build with code-coverage" OFF)

option(USE_PREBUILT_SHADERS "Use externally built HLSL shaders" OFF)

option(NO_WCHAR_T "Use legacy wide-character as unsigned short" OFF)

option(BUILD_FUZZING "Build for fuzz testing" OFF)

# Includes the functions for loading/saving OpenEXR files at runtime
option(ENABLE_OPENEXR_SUPPORT "Build with OpenEXR support" OFF)

# See https://www.ijg.org/
option(ENABLE_LIBJPEG_SUPPORT "Build with libjpeg support" OFF)

# See http://www.libpng.org/pub/png/libpng.html
option(ENABLE_LIBPNG_SUPPORT "Build with libpng support" OFF)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")

if(WINDOWS_STORE OR (DEFINED XBOX_CONSOLE_TARGET))
   set(BUILD_DX12 ON)
   set(BUILD_TOOLS OFF)
   set(BUILD_SAMPLE OFF)
endif()

if((DEFINED XBOX_CONSOLE_TARGET) AND (NOT (XBOX_CONSOLE_TARGET STREQUAL "durango")))
  set(BUILD_DX11 OFF)
endif()

include(GNUInstallDirs)
include(build/CompilerAndLinker.cmake)

if((DEFINED DIRECTX_ARCH) AND (${DIRECTX_ARCH} STREQUAL "arm64ec"))
  # OpenEXR doesn't support ARM64EC
  set(ENABLE_OPENEXR_SUPPORT OFF)
endif()

#--- Library
set(LIBRARY_HEADERS
    DirectXTex/DirectXTex.h
    DirectXTex/DirectXTex.inl)

set(LIBRARY_SOURCES
    DirectXTex/BC.h
    DirectXTex/DDS.h
    DirectXTex/DirectXTexP.h
    DirectXTex/filters.h
    DirectXTex/scoped.h
    DirectXTex/BC.cpp
    DirectXTex/BC4BC5.cpp
    DirectXTex/BC6HBC7.cpp
    DirectXTex/DirectXTexCompress.cpp
    DirectXTex/DirectXTexConvert.cpp
    DirectXTex/DirectXTexDDS.cpp
    DirectXTex/DirectXTexHDR.cpp
    DirectXTex/DirectXTexImage.cpp
    DirectXTex/DirectXTexMipmaps.cpp
    DirectXTex/DirectXTexMisc.cpp
    DirectXTex/DirectXTexNormalMaps.cpp
    DirectXTex/DirectXTexPMAlpha.cpp
    DirectXTex/DirectXTexResize.cpp
    DirectXTex/DirectXTexTGA.cpp
    DirectXTex/DirectXTexUtil.cpp)

if(WIN32)
   list(APPEND LIBRARY_SOURCES
       DirectXTex/DirectXTexFlipRotate.cpp
       DirectXTex/DirectXTexWIC.cpp)
endif()

if(DEFINED XBOX_CONSOLE_TARGET)

    list(APPEND LIBRARY_HEADERS Auxiliary/DirectXTexXbox.h)

    list(APPEND LIBRARY_SOURCES
       Auxiliary/DirectXTexXboxDDS.cpp
       Auxiliary/DirectXTexXboxDetile.cpp
       Auxiliary/DirectXTexXboxImage.cpp
       Auxiliary/DirectXTexXboxTile.cpp)

    if((XBOX_CONSOLE_TARGET STREQUAL "durango") AND BUILD_DX11)
       list(APPEND LIBRARY_SOURCES Auxiliary/DirectXTexXboxD3D11X.cpp)
    endif()

    if(BUILD_DX12)
       list(APPEND LIBRARY_SOURCES Auxiliary/DirectXTexXboxD3D12X.cpp)
    endif()

elseif((BUILD_XBOX_EXTS_XBOXONE OR BUILD_XBOX_EXTS_SCARLETT) AND WIN32)
    if(DEFINED ENV{GameDKLatest})
        cmake_path(SET GameDK_DIR "$ENV{GameDKLatest}")
    endif()
    if(DEFINED ENV{XboxOneXDKLatest})
        cmake_path(SET XboxOneXDK_DIR "$ENV{XboxOneXDKLatest}")
    endif()

    list(APPEND LIBRARY_HEADERS Auxiliary/DirectXTexXbox.h)

    list(APPEND LIBRARY_SOURCES
       Auxiliary/DirectXTexXboxDDS.cpp
       Auxiliary/DirectXTexXboxDetile.cpp
       Auxiliary/DirectXTexXboxImage.cpp
       Auxiliary/DirectXTexXboxTile.cpp)
endif()

if(BUILD_DX11 AND WIN32 AND (NOT (XBOX_CONSOLE_TARGET STREQUAL "durango")))
    set(SHADER_SOURCES
        DirectXTex/Shaders/BC6HEncode.hlsl
        DirectXTex/Shaders/BC7Encode.hlsl)

    list(APPEND LIBRARY_SOURCES
        DirectXTex/BCDirectCompute.h
        DirectXTex/BCDirectCompute.cpp
        DirectXTex/DirectXTexCompressGPU.cpp
        DirectXTex/DirectXTexD3D11.cpp)
endif()

if(BUILD_DX12)
   list(APPEND LIBRARY_SOURCES DirectXTex/DirectXTexD3D12.cpp)

   if(NOT (DEFINED XBOX_CONSOLE_TARGET))
       list(APPEND LIBRARY_SOURCES Common/d3dx12.h)
   endif()
endif()

if(ENABLE_OPENEXR_SUPPORT)
   list(APPEND LIBRARY_HEADERS Auxiliary/DirectXTexEXR.h)
   list(APPEND LIBRARY_SOURCES Auxiliary/DirectXTexEXR.cpp)
endif()

if(ENABLE_LIBJPEG_SUPPORT)
    if(WIN32)
        message(STATUS "Use of the Windows Imaging Component (WIC) instead of libjpeg is recommended.")
    endif()
    list(APPEND LIBRARY_HEADERS Auxiliary/DirectXTexJPEG.h)
    list(APPEND LIBRARY_SOURCES Auxiliary/DirectXTexJPEG.cpp)
endif()

if(ENABLE_LIBPNG_SUPPORT)
    if(WIN32)
        message(STATUS "Use of the Windows Imaging Component (WIC) instead of libpng is recommended.")
    endif()
    list(APPEND LIBRARY_HEADERS Auxiliary/DirectXTexPNG.h)
    list(APPEND LIBRARY_SOURCES Auxiliary/DirectXTexPNG.cpp)
endif()

if(BUILD_DX11 AND WIN32 AND (NOT (XBOX_CONSOLE_TARGET STREQUAL "durango")))
    if(NOT COMPILED_SHADERS)
        if(USE_PREBUILT_SHADERS)
            message(FATAL_ERROR "ERROR: Using prebuilt shaders requires the COMPILED_SHADERS variable is set")
        endif()
        set(COMPILED_SHADERS ${CMAKE_CURRENT_BINARY_DIR}/Shaders/Compiled)
        file(MAKE_DIRECTORY ${COMPILED_SHADERS})
    else()
        file(TO_CMAKE_PATH ${COMPILED_SHADERS} COMPILED_SHADERS)
    endif()

    list(APPEND LIBRARY_SOURCES ${COMPILED_SHADERS}/BC6HEncode_EncodeBlockCS.inc)

    if(NOT USE_PREBUILT_SHADERS)
        find_program(DIRECTX_FXC_TOOL FXC.EXE
          HINTS "C:/Program Files (x86)/Windows Kits/10/bin/${CMAKE_SYSTEM_VERSION}/${DIRECTX_HOST_ARCH}"
                "C:/Program Files (x86)/Windows Kits/10/bin/${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION}/${DIRECTX_HOST_ARCH}")
        message(STATUS "Using LegacyShaderCompiler found in ${DIRECTX_FXC_TOOL}")
        add_custom_command(
            OUTPUT "${COMPILED_SHADERS}/BC6HEncode_EncodeBlockCS.inc"
            MAIN_DEPENDENCY "${PROJECT_SOURCE_DIR}/DirectXTex/Shaders/CompileShaders.cmd"
            DEPENDS ${SHADER_SOURCES}
            COMMENT "Generating HLSL shaders..."
            COMMAND ${CMAKE_COMMAND} -E env CompileShadersOutput="${COMPILED_SHADERS}" $<$<BOOL:${DIRECTX_FXC_TOOL}>:LegacyShaderCompiler=${DIRECTX_FXC_TOOL}> CompileShaders.cmd > "${COMPILED_SHADERS}/compileshaders.log"
            WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/DirectXTex/Shaders"
            USES_TERMINAL)
    endif()
endif()

if(WIN32 AND BUILD_SHARED_LIBS)
  message(STATUS "Build library as a DLL")

  configure_file(
      "${CMAKE_CURRENT_SOURCE_DIR}/build/DirectXTex.rc.in"
      "${CMAKE_CURRENT_BINARY_DIR}/DirectXTex.rc" @ONLY)

  add_library(${PROJECT_NAME} SHARED ${LIBRARY_SOURCES} ${LIBRARY_HEADERS} "${CMAKE_CURRENT_BINARY_DIR}/DirectXTex.rc")

  target_compile_definitions(${PROJECT_NAME} PRIVATE DIRECTX_TEX_EXPORT)
  target_compile_definitions(${PROJECT_NAME} INTERFACE DIRECTX_TEX_IMPORT)

  if(XBOX_CONSOLE_TARGET MATCHES "scarlett")
    target_link_libraries(${PROJECT_NAME} PRIVATE xgameplatform.lib xg_xs.lib xmem.lib)
  elseif(XBOX_CONSOLE_TARGET MATCHES "xboxone")
    target_link_libraries(${PROJECT_NAME} PRIVATE xgameplatform.lib xg_x.lib xmem.lib)
  elseif(XBOX_CONSOLE_TARGET MATCHES "durango")
    target_link_libraries(${PROJECT_NAME} PRIVATE kernelx.lib xg_x.lib combase.lib)
  endif()
else()
  add_library(${PROJECT_NAME} ${LIBRARY_SOURCES} ${LIBRARY_HEADERS})
endif()

if(BUILD_DX11 AND WIN32)
   target_include_directories(${PROJECT_NAME} PRIVATE ${COMPILED_SHADERS})
endif()

source_group(${PROJECT_NAME} REGULAR_EXPRESSION DirectXTex/*.*)

target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/DirectXTex>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_11)

if(BC_USE_OPENMP)
  find_package(OpenMP)
  if(OpenMP_CXX_FOUND)
    target_link_libraries(${PROJECT_NAME} PUBLIC OpenMP::OpenMP_CXX)
  else()
    set(BC_USE_OPENMP OFF)
  endif()
endif()

if(ENABLE_OPENEXR_SUPPORT)
  find_package(OpenEXR REQUIRED)
  target_link_libraries(${PROJECT_NAME} PUBLIC OpenEXR::OpenEXR)
endif()

if(BUILD_XBOX_EXTS_SCARLETT AND WIN32)
    if(EXISTS "${GameDK_DIR}/GXDK/toolKit/include/gxdk.h")
        message(STATUS "Building Xbox extensions for Xbox Series X|S")

        add_library(Xbox::GDKX SHARED IMPORTED)
        set_target_properties(Xbox::GDKX PROPERTIES
            IMPORTED_LOCATION "${GameDK_DIR}/GXDK/bin/Scarlett/xg_xs.dll"
            IMPORTED_IMPLIB "${GameDK_DIR}/GXDK/toolKit/lib/amd64/Scarlett/xg_xs.lib"
            MAP_IMPORTED_CONFIG_MINSIZEREL ""
            MAP_IMPORTED_CONFIG_RELWITHDEBINFO ""
            INTERFACE_COMPILE_DEFINITIONS "_USE_GXDK;_USE_SCARLETT"
            INTERFACE_INCLUDE_DIRECTORIES "${GameDK_DIR}/GXDK/toolKit/include;${GameDK_DIR}/GXDK/toolKit/include/Scarlett")

        target_compile_definitions(${PROJECT_NAME} PRIVATE USE_XBOX_EXTS)
        target_link_libraries(${PROJECT_NAME} PUBLIC Xbox::GDKX)
    else()
        message(FATAL_ERROR "Building Xbox extensions requires GameDKLatest")
    endif()
elseif(BUILD_XBOX_EXTS_XBOXONE AND WIN32)
    if(EXISTS "${GameDK_DIR}/GXDK/toolKit/include/gxdk.h")
        message(STATUS "Building Xbox extensions for XboxOne using the Microsoft GDK")

        add_library(Xbox::GDKX SHARED IMPORTED)
        set_target_properties(Xbox::GDKX PROPERTIES
            IMPORTED_LOCATION "${GameDK_DIR}/GXDK/bin/XboxOne/xg.dll"
            IMPORTED_IMPLIB "${GameDK_DIR}/GXDK/toolKit/lib/amd64/XboxOne/xg.lib"
            MAP_IMPORTED_CONFIG_MINSIZEREL ""
            MAP_IMPORTED_CONFIG_RELWITHDEBINFO ""
            INTERFACE_COMPILE_DEFINITIONS "_USE_GXDK"
            INTERFACE_INCLUDE_DIRECTORIES "${GameDK_DIR}/GXDK/toolKit/include;${GameDK_DIR}/GXDK/toolKit/include/XboxOne")

        target_compile_definitions(${PROJECT_NAME} PRIVATE USE_XBOX_EXTS)
        target_link_libraries(${PROJECT_NAME} PUBLIC Xbox::GDKX)

    elseif(EXISTS "${XboxOneXDK_DIR}/PC/include/xdk.h")
        message(STATUS "Building Xbox extensions for XboxOne using the Xbox One XDK")

        add_library(Xbox::XDK SHARED IMPORTED)
        set_target_properties(Xbox::XDK PROPERTIES
            IMPORTED_LOCATION "${XboxOneXDK_DIR}/bin/xg.dll"
            IMPORTED_IMPLIB "${XboxOneXDK_DIR}/PC/lib/amd64/xg.lib"
            MAP_IMPORTED_CONFIG_MINSIZEREL ""
            MAP_IMPORTED_CONFIG_RELWITHDEBINFO ""
            INTERFACE_INCLUDE_DIRECTORIES "${XboxOneXDK_DIR}/PC/include")

        target_compile_definitions(${PROJECT_NAME} PRIVATE USE_XBOX_EXTS)
        target_link_libraries(${PROJECT_NAME} PUBLIC Xbox::XDK)
    else()
        message(FATAL_ERROR "Building Xbox extensions requires GameDKLatest or XboxOneXDKLatest")
    endif()
endif()

if(ENABLE_LIBJPEG_SUPPORT)
  find_package(JPEG REQUIRED)
  target_link_libraries(${PROJECT_NAME} PUBLIC JPEG::JPEG)
endif()

if(ENABLE_LIBPNG_SUPPORT)
  find_package(PNG REQUIRED)
  target_link_libraries(${PROJECT_NAME} PUBLIC PNG::PNG)
endif()

if(NOT MINGW)
    target_precompile_headers(${PROJECT_NAME} PRIVATE DirectXTex/DirectXTexP.h)
endif()

if(MINGW OR (NOT WIN32))
    find_package(directxmath CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PUBLIC Microsoft::DirectXMath)

    find_package(directx-headers CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PUBLIC Microsoft::DirectX-Headers)
    target_compile_definitions(${PROJECT_NAME} PUBLIC USING_DIRECTX_HEADERS)
else()
    find_package(directxmath CONFIG QUIET)
    find_package(directx-headers CONFIG QUIET)
endif()

if(directxmath_FOUND)
    message(STATUS "Using DirectXMath package")
    target_link_libraries(${PROJECT_NAME} PRIVATE Microsoft::DirectXMath)
endif()

if(directx-headers_FOUND)
    message(STATUS "Using DirectX-Headers package")
    target_link_libraries(${PROJECT_NAME} PRIVATE Microsoft::DirectX-Headers)
    target_compile_definitions(${PROJECT_NAME} PRIVATE USING_DIRECTX_HEADERS)
    target_compile_options(${PROJECT_NAME} PRIVATE $<$<CXX_COMPILER_ID:MSVC,Intel>:/wd4062> $<$<CXX_COMPILER_ID:Clang,IntelLLVM>:-Wno-switch-enum>)
endif()

#--- Package
include(CMakePackageConfigHelpers)

string(TOLOWER ${PROJECT_NAME} PACKAGE_NAME)

write_basic_package_version_file(
  ${PACKAGE_NAME}-config-version.cmake
  VERSION ${DIRECTXTEX_VERSION}
  COMPATIBILITY AnyNewerVersion)

install(TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}-targets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/build/${PROJECT_NAME}-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}-config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PACKAGE_NAME})

install(EXPORT ${PROJECT_NAME}-targets
  FILE ${PROJECT_NAME}-targets.cmake
  NAMESPACE Microsoft::
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PACKAGE_NAME})

install(FILES ${LIBRARY_HEADERS}
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}-config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}-config-version.cmake
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/${PACKAGE_NAME})

# Create pkg-config file
include(build/JoinPaths.cmake)
# from: https://github.com/jtojnar/cmake-snips#concatenating-paths-when-building-pkg-config-files
join_paths(DIRECTXTEX_INCLUDEDIR_FOR_PKG_CONFIG "\${prefix}" "${CMAKE_INSTALL_INCLUDEDIR}")
join_paths(DIRECTXTEX_LIBDIR_FOR_PKG_CONFIG "\${prefix}"     "${CMAKE_INSTALL_LIBDIR}")

set(DIRECTXTEX_DEP_L "")
if(ENABLE_OPENEXR_SUPPORT)
  list(APPEND DIRECTXTEX_DEP_L "OpenEXR")
endif()
if(directxmath_FOUND)
  list(APPEND DIRECTXTEX_DEP_L "DirectXMath")
endif()
if(directx-headers_FOUND)
  list(APPEND DIRECTXTEX_DEP_L "DirectX-Headers")
endif()
if(ENABLE_LIBJPEG_SUPPORT AND JPEG_FOUND)
  list(APPEND DIRECTXTEX_DEP_L "libjpeg")
endif()
if(ENABLE_LIBPNG_SUPPORT AND PNG_FOUND)
  list(APPEND DIRECTXTEX_DEP_L "libpng")
endif()

list(LENGTH DIRECTXTEX_DEP_L DEP_L)
if(DEP_L)
  STRING(REPLACE ";" ", " DIRECTXTEX_DEP " ${DIRECTXTEX_DEP_L}")
endif()

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/build/DirectXTex.pc.in"
    "${CMAKE_CURRENT_BINARY_DIR}/DirectXTex.pc" @ONLY)

# Install the pkg-config file
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/DirectXTex.pc"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

#--- Command-line tools
set(TOOL_EXES "")

if(BUILD_TOOLS AND WIN32)
  add_executable(texassemble
    Texassemble/texassemble.cpp
    Texassemble/texassemble.rc
    Common/CmdLineHelpers.h
    Common/settings.manifest
    Texassemble/AnimatedGif.cpp)
  target_compile_features(texassemble PRIVATE cxx_std_17)
  target_link_libraries(texassemble PRIVATE ${PROJECT_NAME} ole32.lib version.lib)
  source_group(texassemble REGULAR_EXPRESSION Texassemble/*.*)
  list(APPEND TOOL_EXES texassemble)
endif()

if(BUILD_TOOLS AND BUILD_DX11 AND WIN32)
  add_executable(texconv
    Texconv/texconv.cpp
    Texconv/texconv.rc
    Common/CmdLineHelpers.h
    Common/settings.manifest
    Texconv/ExtendedBMP.cpp
    Texconv/PortablePixMap.cpp)
  target_compile_features(texconv PRIVATE cxx_std_17)
  target_link_libraries(texconv PRIVATE ${PROJECT_NAME} ole32.lib shell32.lib version.lib)
  source_group(texconv REGULAR_EXPRESSION Texconv/*.*)
  list(APPEND TOOL_EXES texconv)
endif()

if(BUILD_TOOLS AND WIN32)
  add_executable(texdiag
    Texdiag/texdiag.cpp
    Texdiag/texdiag.rc
    Common/CmdLineHelpers.h
    Common/settings.manifest)
  target_compile_features(texdiag PRIVATE cxx_std_17)
  target_link_libraries(texdiag PRIVATE ${PROJECT_NAME} ole32.lib version.lib)
  source_group(texdiag REGULAR_EXPRESSION Texdiag/*.*)
  list(APPEND TOOL_EXES texdiag)
endif()

foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
  target_include_directories(${t} PRIVATE Common)
endforeach()

if(BUILD_TOOLS AND WIN32)
  if(ENABLE_OPENEXR_SUPPORT)
    foreach(t IN LISTS TOOL_EXES)
      target_include_directories(${t} PRIVATE Auxiliary)
      target_link_libraries(${t} PRIVATE ${OPENEXR_ILMIMF_LIBRARY})
      target_compile_definitions(${t} PRIVATE USE_OPENEXR)
    endforeach()
  endif()
  if(ENABLE_LIBJPEG_SUPPORT)
    foreach(t IN LISTS TOOL_EXES)
      target_include_directories(${t} PRIVATE Auxiliary)
      target_link_libraries(${t} PRIVATE JPEG::JPEG)
      target_compile_definitions(${t} PRIVATE USE_LIBJPEG)
    endforeach()
  endif()
  if(ENABLE_LIBPNG_SUPPORT)
    foreach(t IN LISTS TOOL_EXES)
      target_include_directories(${t} PRIVATE Auxiliary)
      target_link_libraries(${t} PRIVATE PNG::PNG)
      target_compile_definitions(${t} PRIVATE USE_LIBPNG)
    endforeach()
  endif()
  if(BUILD_XBOX_EXTS_SCARLETT OR BUILD_XBOX_EXTS_XBOXONE)
    target_include_directories(texconv PRIVATE Auxiliary)
    target_compile_definitions(texconv PRIVATE USE_XBOX_EXTS)
    target_link_libraries(texconv PUBLIC $<TARGET_NAME_IF_EXISTS:Xbox::GDKX> $<TARGET_NAME_IF_EXISTS:Xbox::XDK>)

    if(${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.21")
      add_custom_command(TARGET texconv POST_BUILD
          COMMAND ${CMAKE_COMMAND} -E copy
          $<TARGET_RUNTIME_DLLS:texconv> $<TARGET_FILE_DIR:texconv>
          COMMAND_EXPAND_LISTS
          )
    endif()
  endif()
endif()

#--- DDSView sample
if(BUILD_SAMPLE AND BUILD_DX11 AND WIN32)
  list(APPEND TOOL_EXES ddsview)

  add_executable(ddsview WIN32
    DDSView/ddsview.cpp
    DDSView/ddsview.rc
    ${COMPILED_SHADERS}/ddsview_ps1D.inc)
  target_link_libraries(ddsview PRIVATE ${PROJECT_NAME} d3d11.lib ole32.lib)
  source_group(ddsview REGULAR_EXPRESSION DDSView/*.*)

  target_include_directories(ddsview PRIVATE ${COMPILED_SHADERS})

  if(NOT USE_PREBUILT_SHADERS)
      add_custom_command(
            OUTPUT "${COMPILED_SHADERS}/ddsview_ps1D.inc"
            MAIN_DEPENDENCY "${PROJECT_SOURCE_DIR}/DDSView/hlsl.cmd"
            DEPENDS "DDSView/ddsview.hlsl"
            COMMENT "Generating HLSL shaders for DDSView..."
            COMMAND ${CMAKE_COMMAND} -E env CompileShadersOutput="${COMPILED_SHADERS}" $<$<BOOL:${DIRECTX_FXC_TOOL}>:LegacyShaderCompiler=${DIRECTX_FXC_TOOL}> hlsl.cmd > "${COMPILED_SHADERS}/hlsl_ddsview.log"
            WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/DDSView"
            USES_TERMINAL)
  endif()
endif()

if(directxmath_FOUND)
  foreach(t IN LISTS TOOL_EXES)
    target_link_libraries(${t} PRIVATE Microsoft::DirectXMath)
  endforeach()
endif()

if(TOOL_EXES)
  message(STATUS "Building tools: ${TOOL_EXES}")
endif()

if(MSVC)
    foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
      target_compile_options(${t} PRIVATE /Wall /GR-)
    endforeach()

    if(NO_WCHAR_T)
      message(STATUS "Using non-native wchar_t as unsigned short")
      foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
        target_compile_options(${t} PRIVATE "/Zc:wchar_t-")
      endforeach()
    endif()
endif()

foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
  target_compile_definitions(${t} PRIVATE ${COMPILER_DEFINES})
  target_compile_options(${t} PRIVATE ${COMPILER_SWITCHES})
  target_link_options(${t} PRIVATE ${LINKER_SWITCHES})
endforeach()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|IntelLLVM")
    set(WarningsLib -Wall -Wpedantic -Wextra)

    if((BUILD_XBOX_EXTS_XBOXONE OR BUILD_XBOX_EXTS_SCARLETT) AND WIN32)
        list(APPEND WarningsLib "-Wno-microsoft-enum-value" "-Wno-non-virtual-dtor" "-Wno-ignored-pragmas" "-Wno-deprecated-dynamic-exception-spec")
        if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 13.0)
            list(APPEND WarningsLib "-Wno-reserved-identifier")
        endif()
    endif()

    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 16.0)
        list(APPEND WarningsLib "-Wno-unsafe-buffer-usage")
    endif()
    target_compile_options(${PROJECT_NAME} PRIVATE ${WarningsLib})

    set(WarningsEXE ${WarningsLib} "-Wno-c++98-compat" "-Wno-c++98-compat-pedantic" "-Wno-switch" "-Wno-switch-enum" "-Wno-switch-default" "-Wno-covered-switch-default" "-Wno-language-extension-token" "-Wno-missing-prototypes" "-Wno-global-constructors" "-Wno-double-promotion")
    foreach(t IN LISTS TOOL_EXES)
      target_compile_options(${t} PRIVATE ${WarningsEXE})
    endforeach()
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
      target_compile_options(${t} PRIVATE "-Wno-ignored-attributes" "-Walloc-size-larger-than=4GB")

      if(BUILD_SHARED_LIBS)
        target_compile_options(${t} PRIVATE "-Wno-attributes")
      endif()
    endforeach()
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Intel")
    set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD 14)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")

    if((CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.37)
       AND (NOT (${DIRECTX_ARCH} MATCHES "^arm"))
       AND ((${DIRECTX_ARCH} STREQUAL "x64") OR (CMAKE_SIZEOF_VOID_P EQUAL 8)))
       # Enable since DirectXTex library has a lot of large switch statements
      target_compile_options(${PROJECT_NAME} PRIVATE /jumptablerdata)
    endif()

    if(ENABLE_CODE_ANALYSIS)
      message(STATUS "Building with Code Analysis (PREFIX)")
      foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
        target_compile_options(${t} PRIVATE /analyze /WX)
      endforeach()
    endif()

    if(ENABLE_SPECTRE_MITIGATION
       AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.13)
       AND (NOT ENABLE_OPENEXR_SUPPORT)
       AND (NOT WINDOWS_STORE))
        message(STATUS "Building Spectre-mitigated libraries")
        foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
          target_compile_options(${t} PRIVATE "/Qspectre")
        endforeach()
    endif()

   if((CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.26)
      AND BUILD_XBOX_EXTS_XBOXONE)
        foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
          target_compile_options(${t} PRIVATE /wd5104 /wd5204)
        endforeach()
    endif()

    set(WarningsEXE "/wd4061" "/wd4062" "/wd4365" "/wd4514" "/wd4571" "/wd4625" "/wd4626" "/wd4627" "/wd4668" "/wd4710" "/wd4711" "/wd4751" "/wd4774" "/wd4820" "/wd5026" "/wd5027" "/wd5039" "/wd5045" "/wd5219")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.34)
      list(APPEND WarningsEXE "/wd5262" "/wd5264")
    endif()
    foreach(t IN LISTS TOOL_EXES)
      target_compile_options(${t} PRIVATE ${WarningsEXE})
    endforeach()

    if(BUILD_FUZZING AND (NOT WINDOWS_STORE))
      string(REPLACE "/DNDEBUG" "" CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE})
      string(REPLACE "/DNDEBUG" "" CMAKE_CXX_FLAGS_RELWITHDEBINFO ${CMAKE_CXX_FLAGS_RELWITHDEBINFO})

      if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.32)
        foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
          target_compile_options(${t} PRIVATE ${ASAN_SWITCHES})
          target_link_libraries(${t} PRIVATE ${ASAN_LIBS})
        endforeach()
      endif()
    endif()
endif()

if(WIN32)
    if(XBOX_CONSOLE_TARGET STREQUAL "durango")
        set(WINVER 0x0602)
    elseif(BUILD_DX12 OR (${DIRECTX_ARCH} MATCHES "^arm64"))
        message(STATUS "Building with DirectX 12 Runtime support")
        set(WINVER 0x0A00)
    elseif(${DIRECTX_ARCH} MATCHES "^arm")
        set(WINVER 0x0602)
    else()
        message(STATUS "Building with Windows 8.1 compatibility")
        set(WINVER 0x0603)
    endif()

    foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
      target_compile_definitions(${t} PRIVATE _WIN32_WINNT=${WINVER})
    endforeach()

    if(DISABLE_MSVC_ITERATOR_DEBUGGING)
      foreach(t IN LISTS TOOL_EXES ITEMS ${PROJECT_NAME})
        target_compile_definitions(${t} PRIVATE _ITERATOR_DEBUG_LEVEL=0)
      endforeach()
    endif()
endif()

if(BUILD_TOOLS AND BUILD_DX11 AND WIN32)
    set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT texconv)
endif()

if(BUILD_TOOLS AND (NOT VCPKG_TOOLCHAIN))
    foreach(t IN LISTS TOOL_EXES)
      install(TARGETS ${t} RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
    endforeach()
endif()

#--- Test suite
if(WIN32 AND (NOT WINDOWS_STORE) AND (NOT (DEFINED XBOX_CONSOLE_TARGET)))
    include(CTest)
    if(BUILD_TESTING AND (EXISTS "${CMAKE_CURRENT_LIST_DIR}/Tests/CMakeLists.txt"))
        enable_testing()
        add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/Tests)

        if(ENABLE_CODE_COVERAGE AND (DEFINED COV_COMPILER_SWITCHES))
          target_compile_options(${PROJECT_NAME} PRIVATE ${COV_COMPILER_SWITCHES})
        endif()
    elseif(BUILD_FUZZING AND (EXISTS "${CMAKE_CURRENT_LIST_DIR}/Tests/fuzzloaders/CMakeLists.txt"))
        message(STATUS "Building for fuzzing")
        add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/Tests/fuzzloaders)
    endif()
endif()
