cmake_minimum_required(VERSION 3.5...3.25)
#####################################################################################
# offer the choice of having nvpro_core as a sub-folder... good for packaging a sample
#
# if BASE_DIRECTORY not defined, it means this cmake file was called as the first entry point and not included
if(NOT BASE_DIRECTORY) # if not defined, it means this cmake file was called as the first entry point and not included
  message(FATAL_ERROR "cannot be directly invoked")
else()
  Message(STATUS "-------------------------------")
  Message(STATUS "Processing nvpro_core files")
endif()


set(NVPWINDOW_SOURCE
  ${BASE_DIRECTORY}/nvpro_core/nvp/nvpwindow.cpp
  ${BASE_DIRECTORY}/nvpro_core/nvp/nvpwindow.hpp
  ${BASE_DIRECTORY}/nvpro_core/nvp/nvpsystem.cpp
  ${BASE_DIRECTORY}/nvpro_core/nvp/nvpsystem.hpp
  ${BASE_DIRECTORY}/nvpro_core/nvp/nvpfilesystem.cpp
  ${BASE_DIRECTORY}/nvpro_core/nvp/nvpfilesystem.hpp
  # Do not add project_name.cpp here as it contains per-project dependent data.
  # perproject_globals.cpp will be automatically added to the per-project builds via
  # COMMON_SOURCE_FILES
  ${BASE_DIRECTORY}/nvpro_core/nvp/perproject_globals.hpp
)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3")

if(UNIX) 
  LIST(APPEND NVPWINDOW_SOURCE ${BASE_DIRECTORY}/nvpro_core/nvp/nvpsystem_linux.cpp )
else()
  LIST(APPEND NVPWINDOW_SOURCE ${BASE_DIRECTORY}/nvpro_core/nvp/nvpsystem_win32.cpp )
endif()

if(NOT (CMAKE_SIZEOF_VOID_P EQUAL 8))
  message( FATAL_ERROR "64-bit builds are mandatory for this framework" )
endif()


# Add third party libraries. This sets THIRDPARTY_LIBRARIES to list of added libraries
add_subdirectory(third_party)


# add packages that may be needed by some of the samples
_optional_package_VulkanSDK()
_optional_package_ShaderC()
_optional_package_OpenGL()
_optional_package_DirectX12()
_optional_package_Cuda()
_optional_package_NVML()
_optional_package_NVToolsExt()

# process the rest of some cmake code that needs to be done after the packages
_process_shared_cmake_code()

file(GLOB FILEFORMATS_SOURCE fileformats/*.cpp fileformats/*.hpp fileformats/*.inl fileformats/*.h)
file(GLOB NVMATH_SOURCE nvmath/*.cpp nvmath/*.hpp nvmath/*.inl nvmath/*.h)
file(GLOB NVHELPERS_SOURCE nvh/*.cpp nvh/*.hpp nvh/*.inl nvh/*.h)

if (NOT TARGET tinygltf)
  list(REMOVE_ITEM NVHELPERS_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/nvh/gltfscene.cpp)
  list(REMOVE_ITEM NVHELPERS_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/nvh/gltfscene.hpp)
endif()

set(library_name "nvpro_core")

if(USING_OPENGL)
  file(GLOB NVHELPERSGL_SOURCE nvgl/*.cpp nvgl/*.hpp nvgl/*.inl nvgl/*.h)
  source_group("nvgl" FILES ${NVHELPERSGL_SOURCE})
  
  string(CONCAT library_name ${library_name} "_gl")
endif()

if(USING_VULKANSDK)
  file(GLOB NVHELPERSVK_SOURCE nvvk/*.cpp nvvk/*.hpp nvvk/*.inl nvvk/*.h)

  set(GENERATED_EXTENSION_DIR "${CMAKE_CURRENT_BINARY_DIR}/extensions_vk_gen")
  list(APPEND NVHELPERSVK_SOURCE
    "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.cpp"
    "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.hpp"
    )

  # the implementation depends on SUPPORTS_AFTERMATH and thus can only be provided
  # as part of the individual sample projects. Check setup.cmake where we insert
  # nsight_aftermath_vk.cpp into the projects automatically
  list(REMOVE_ITEM NVHELPERSVK_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/nvvk/nsight_aftermath_vk.cpp)

  # Adding compile_glsl 
  if(EXISTS ${BASE_DIRECTORY}/nvpro_core/cmake/utilities.cmake)
    include(${BASE_DIRECTORY}/nvpro_core/cmake/utilities.cmake)
  endif()

  # Vulkan-high-level helpers
  file(GLOB NVVKHL_SOURCE nvvkhl/*.cpp nvvkhl/*.hpp nvvkl/*.inl nvvkhl/*.h)
  set(SHD_DIR nvvkhl/shaders)
  file(GLOB SHD_HDR  ${SHD_DIR}/*.h)
  file(GLOB SHD_SRC ${SHD_DIR}/*.glsl)
  # Compiling shaders to SPIR-V header
  compile_glsl(
      SOURCE_FILES ${SHD_SRC}
      HEADER_FILES ${SHD_HDR}
      DST "${CMAKE_CURRENT_SOURCE_DIR}/_autogen" 
      VULKAN_TARGET "vulkan1.3"
      HEADER ON
      DEPENDENCY ${Vulkan_BUILD_DEPENDENCIES}
      FLAGS "-I${SHD_DIR}" "-I${CMAKE_CURRENT_SOURCE_DIR}" -g
      )

  if (NOT TARGET vma)
    list(REMOVE_ITEM NVHELPERSVK_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/nvvk/memallocator_vma_vk.hpp)
    list(REMOVE_ITEM NVHELPERSVK_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/nvvk/memallocator_vma_vk.inl)
    list(REMOVE_ITEM NVVKHL_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/nvvkhl/alloc_vma.hpp)
  endif()
  
  source_group("nvvk" FILES ${NVHELPERSVK_SOURCE})
  source_group("nvvkhl" FILES ${NVVKHL_SOURCE})
  source_group("nvvkhl/shaders" FILES ${GLSL_SOURCES} ${GLSL_HEADERS})
  set(NVVKHL_SHADERS ${GLSL_SOURCES} ${GLSL_HEADERS})

  string(CONCAT library_name ${library_name} "_vk")
  
  if(WIN32)
    add_definitions(-DVK_USE_PLATFORM_WIN32_KHR)
  endif(WIN32)
  if(UNIX)
    add_definitions(-DVK_USE_PLATFORM_XCB_KHR)
    add_definitions(-DVK_USE_PLATFORM_XLIB_KHR)
  endif(UNIX)
  
  # Some samples expect find_package(Python ...) to have been called already
  find_package(Python COMPONENTS Interpreter)

  # The following checks if Python is available and run the script to generate the Vulkan extensions
  # using the version of Vulkan SDK installed on the system
  option(Vulkan_generate_extensions "Regenerate nvvk/extensions_vk.cpp from the currently installed Vulkan SDK version at configuration time." ON)
  set(Vulkan_EXTENSIONS_GENERATED FALSE)
  if(Vulkan_generate_extensions)
    find_package(Python3 COMPONENTS Interpreter)
    if(Python3_FOUND)
      find_file(VULKAN_XML vk.xml
        PATHS "$ENV{VULKAN_SDK}" "/usr"
        PATH_SUFFIXES "share/vulkan/registry"
        REQUIRED NO_DEFAULT_PATH)
      add_custom_command(
        OUTPUT "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.cpp" "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.hpp"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/nvvk/extensions_vk.py" "${VULKAN_XML}"
        COMMAND ${Python3_EXECUTABLE} "${CMAKE_CURRENT_SOURCE_DIR}/nvvk/extensions_vk.py" "${VULKAN_XML}" --output-dir "${GENERATED_EXTENSION_DIR}/nvvk" --beta
        COMMENT "Vulkan extension code generation with nvvk/extensions_vk.py (cmake option: Vulkan_generate_extensions)"
      )
      set(Vulkan_EXTENSIONS_GENERATED TRUE)
    else()
      message(WARNING "Python3 not found. Cannot generate code with local Vulkan SDK. Falling back to packaged code.")
    endif()
  endif(Vulkan_generate_extensions)

  # Use fallback source files
  if(NOT Vulkan_EXTENSIONS_GENERATED)
    add_custom_command(
      OUTPUT "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.cpp"
      COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_CURRENT_SOURCE_DIR}/nvvk/extensions_vk.cpp.in" "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.cpp"
      COMMENT "Copying fallback vulkan extension code (disabled or python3 not found)"
    )
    add_custom_command(
      OUTPUT "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.hpp"
      COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_CURRENT_SOURCE_DIR}/nvvk/extensions_vk.hpp.in" "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.hpp"
      COMMENT "Copying fallback vulkan extension code (disabled or python3 not found)"
    )
  endif()
else()
  # KTX files can only be loaded if Vulkan headers are available.
  # Another option here would be to only add nv_ktx's files if _add_package_KTX()
  # is called. Currently, _add_package_KTX() adds optional supercompression dependencies.
  list(REMOVE_ITEM FILEFORMATS_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/fileformats/nv_ktx.h)
  list(REMOVE_ITEM FILEFORMATS_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/fileformats/nv_ktx.cpp)
endif(USING_VULKANSDK)

if(USING_IMGUI)
  set(IMGUI_SOURCE 
      imgui/imgui_axis.cpp  
      imgui/imgui_axis.hpp
      imgui/imgui_camera_widget.cpp 
      imgui/imgui_camera_widget.h 
      imgui/imgui_helper.cpp
      imgui/imgui_helper.h
      imgui/imgui_icon.cpp
      imgui/imgui_icon.h
      imgui/imgui_orient.cpp  
      imgui/imgui_orient.h
      imgui/imgui_handler.cpp  
      imgui/imgui_handler.h
      ${IMGUI_DIR}/backends/imgui_impl_glfw.cpp
      ${IMGUI_DIR}/backends/imgui_impl_glfw.h
      )
  if (NOT TARGET tinygltf)
    # depends on json.hpp from tinygltf
    list(REMOVE_ITEM IMGUI_SOURCE imgui/imgui_camera_widget.h)
    list(REMOVE_ITEM IMGUI_SOURCE imgui/imgui_camera_widget.cpp)
  endif()

  if(USING_OPENGL)
    # replaced with our own copy
    list(APPEND IMGUI_SOURCE 
         ${CMAKE_CURRENT_SOURCE_DIR}/imgui/backends/imgui_impl_gl.cpp
         ${CMAKE_CURRENT_SOURCE_DIR}/imgui/backends/imgui_impl_gl.h
         )
  endif() 
 
  if(USING_VULKANSDK)
    list(APPEND IMGUI_SOURCE 
         ${CMAKE_CURRENT_SOURCE_DIR}/imgui/backends/imgui_vk_extra.cpp 
         ${CMAKE_CURRENT_SOURCE_DIR}/imgui/backends/imgui_vk_extra.h
         ${IMGUI_DIR}/backends/imgui_impl_vulkan.cpp 
         ${IMGUI_DIR}/backends/imgui_impl_vulkan.h)
  endif()
 
  if(USING_DIRECTX12)
    list(APPEND IMGUI_SOURCE 
         ${IMGUI_DIR}/backends/imgui_impl_dx12.cpp
         ${IMGUI_DIR}/backends/imgui_impl_dx12.h)
  endif()

endif()

if (USING_DIRECTX12)
  file(GLOB NVHELPERSDX12_SOURCE nvdx12/*.cpp nvdx12/*.hpp nvdx12/*.inl nvdx12/*.h)
  source_group("nvdx12" FILES ${NVHELPERSDX12_SOURCE})

  string(CONCAT library_name ${library_name} "_dx12")
endif()


if (USING_OPTIX)
  Message(STATUS "Note: adding Optix utility files")
  file(GLOB NVHELPERSOPTIX_FILES nvoptix/*.cpp nvoptix/*.h)
  source_group("nvoptix" FILES ${NVHELPERSOPTIX_FILES})
  
  string(CONCAT library_name ${library_name} "_optix")
endif()

if (USING_SHADERC)
  string(CONCAT library_name ${library_name} "_shaderc")
endif()

source_group("fileformats" FILES ${FILEFORMATS_SOURCE})
source_group("noise" FILES ${NOISE_SOURCE})
source_group("nvmath" FILES ${NVMATH_SOURCE})
source_group("nvh" FILES ${NVHELPERS_SOURCE})
source_group("nvp" FILES ${NVPWINDOW_SOURCE})
source_group("imgui" FILES ${IMGUI_SOURCE})

if(MSVC)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif(MSVC)


if(USING_OPENGL)
  set(OPENGL_FILES
    ${has_GL_FILES} 
  )
endif()

add_library(nvpro_core STATIC 
  ${NVHELPERS_SOURCE}
  ${NVMATH_SOURCE}
  ${IMGUI_SOURCE}
  ${FILEFORMATS_SOURCE}
  ${NVPWINDOW_SOURCE}
  ${OPENGL_FILES}
  ${NVHELPERSVK_SOURCE}
  ${NVVKHL_SOURCE}
  ${NVVKHL_SHADERS}
  ${NVHELPERSGL_SOURCE}
  ${NVHELPERSDX12_SOURCE}
  ${PACKAGE_SOURCE_FILES}
  ${NVHELPERSOPTIX_FILES}
)

target_compile_features(nvpro_core PUBLIC cxx_std_20)

# Create precompiled header (added in CMake 3.16)
if(${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.16.0")
  set(_PCH_FILES
    "<algorithm>"
    ${CMAKE_CURRENT_SOURCE_DIR}/nvh/nvprint.hpp
  )
  target_precompile_headers(nvpro_core PRIVATE ${_PCH_FILES})
endif()

# Add X11 for screenshot support on linux
if(UNIX)
  find_package(X11 REQUIRED)
  LIST(APPEND PLATFORM_LIBRARIES ${X11_LIBRARIES})
endif()

# Make Windows compilation somewhat faster, and link with Dbghelp for
# stack trace information
if(WIN32)
  target_compile_definitions(nvpro_core PRIVATE WIN32_LEAN_AND_MEAN)
  LIST(APPEND PLATFORM_LIBRARIES DbgHelp)
endif()

if(USING_IMGUI)
  # Enable ImVec* operators for all nvpro_core. See imgui.h.
  target_compile_definitions(nvpro_core PRIVATE IMGUI_DEFINE_MATH_OPERATORS)
endif()

if(USING_VULKANSDK)
  target_include_directories(nvpro_core PUBLIC "${GENERATED_EXTENSION_DIR}")
  option(Vulkan_update_codegen_fallback "Update the nvvk/extensions_vk.cpp fallback code in the source directory." OFF)
  if(Vulkan_EXTENSIONS_GENERATED AND Vulkan_update_codegen_fallback)
    add_custom_command(
      TARGET nvpro_core POST_BUILD
      DEPENDS "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.cpp" "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.hpp"
      COMMAND ${CMAKE_COMMAND} -E copy_if_different "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.cpp" "${CMAKE_CURRENT_SOURCE_DIR}/nvvk/extensions_vk.cpp.in"
      COMMAND ${CMAKE_COMMAND} -E copy_if_different "${GENERATED_EXTENSION_DIR}/nvvk/extensions_vk.hpp" "${CMAKE_CURRENT_SOURCE_DIR}/nvvk/extensions_vk.hpp.in"
      COMMENT "Updating vulkan extension code generation fallback"
    )
  endif()
endif()

if(USING_ZLIB)
  target_link_libraries(nvpro_core zlibstatic)
endif()

# zstd and basisu are always enabled together and basisu links with zstd, so no
# need to check for them separately
if(USING_BASISU)
  target_link_libraries(nvpro_core basisu)
endif()

# Libstdc++ and libc++'s std::execution::par use TBB.
if(UNIX)
  find_package(TBB)
  if(TBB_FOUND)
    target_link_libraries(nvpro_core TBB::tbb)
  else()
    message(FATAL_ERROR "Could not find TBB. Try installing the TBB development \
library using your package manager: `sudo apt install libtbb-dev` on Ubuntu, \
`sudo dnf install tbb-devel` on Fedora, `sudo pacman -S onetbb` on Arch, or on \
NixOS, add `tbb` to your environment.")
  endif()
endif()

target_link_libraries(nvpro_core
  ${PLATFORM_LIBRARIES} 
  glfw 
  imgui
  implot
  fmt
  glm
  thread_pool
  ${THIRDPARTY_LIBRARIES}
 )

set_target_properties(nvpro_core PROPERTIES OUTPUT_NAME ${library_name})
_set_target_output(nvpro_core)

message(STATUS "nvpro_core library name: ${library_name}")

