cmake_minimum_required(VERSION 3.20)

project(EAStdC LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

option(EA_FETCH_DEPS "Fetch missing EA dependencies from GitHub via FetchContent" ON)

set(EA_GIT_BASE_URL "https://github.com" CACHE STRING "Base URL for EA dependency repositories")
set(EA_GITHUB_ORG "DOCaCola" CACHE STRING "GitHub org/user for EA package repositories")
set(EA_GIT_TAG "main" CACHE STRING "Git tag/branch/commit for EA package repositories")

function(ea_apply_msvc_options tgt)
  if(MSVC)
    target_compile_definitions(${tgt} PRIVATE _CRT_SECURE_NO_WARNINGS)
    target_compile_options(${tgt} PRIVATE
      /MP
      /W4
      $<$<CONFIG:RELEASE>:-Zo -d2SSAOptimizer->
      -bigobj /FS /Zf
      -w44946 -w44555 -w44312 -w44296 -w44287 -w44265 -w44254 -w44242
      /wd4738 /wd4548 /wd4619 /wd4530 /wd4217 /wd4574 /wd4365 /wd4100 /wd4324 /wd4350 /wd4464 /wd4482 /wd4577 /wd4711 /wd4746 /wd4986 /wd4987 /wd4819
    )
  endif()
endfunction()

function(ea_apply_common_config_defines tgt)
  target_compile_definitions(${tgt} PRIVATE
    $<$<CONFIG:RELEASE>:NDEBUG>
    $<$<CONFIG:DEBUG>:EA_DEBUG>
    $<$<CONFIG:RelWithDebInfo>:EA_DEBUG>
  )
endfunction()

function(ea_repo_url out_var repo_name)
  set(${out_var} "${EA_GIT_BASE_URL}/${EA_GITHUB_ORG}/${repo_name}.git" PARENT_SCOPE)
endfunction()

function(ea_add_dep name)
  if(TARGET ${name})
    return()
  endif()

  if(NOT EA_FETCH_DEPS)
    message(FATAL_ERROR "Missing dependency target '${name}'. Provide it in the parent project or set EA_FETCH_DEPS=ON.")
  endif()

  string(TOUPPER "${name}" name_upper)
  set(repo_var "EA_${name_upper}_GIT_REPOSITORY")
  set(tag_var  "EA_${name_upper}_GIT_TAG")

  if(NOT DEFINED ${repo_var})
    ea_repo_url(default_repo "${name}")
    set(${repo_var} "${default_repo}" CACHE STRING "Git repository for ${name}")
  endif()
  if(NOT DEFINED ${tag_var})
    set(${tag_var} "${EA_GIT_TAG}" CACHE STRING "Git tag/branch/commit for ${name}")
  endif()

  FetchContent_Declare(
    ${name}
    GIT_REPOSITORY "${${repo_var}}"
    GIT_TAG "${${tag_var}}"
    GIT_SHALLOW TRUE
    GIT_SUBMODULES ""
  )
  FetchContent_MakeAvailable(${name})
endfunction()

ea_add_dep(EABase)
ea_add_dep(EAThread)
ea_add_dep(EAAssert) # required for EAStdC in EAWebKit support packages

file(GLOB eastdc_sources CONFIGURE_DEPENDS "${CMAKE_CURRENT_LIST_DIR}/source/*.cpp")
file(GLOB_RECURSE eastdc_headers CONFIGURE_DEPENDS "${CMAKE_CURRENT_LIST_DIR}/include/*.h")

add_library(EAStdC ${eastdc_sources})
target_sources(EAStdC PRIVATE ${eastdc_headers})

target_include_directories(EAStdC PUBLIC "${CMAKE_CURRENT_LIST_DIR}/include")
target_include_directories(EAStdC PRIVATE "${CMAKE_CURRENT_LIST_DIR}/source")

target_link_libraries(EAStdC PUBLIC EABase EAThread EAAssert)

target_compile_definitions(EAStdC PRIVATE
  _GLIBCPP_USE_C99=1
  _GLIBCXX_USE_C99_MATH=1
)

ea_apply_common_config_defines(EAStdC)
ea_apply_msvc_options(EAStdC)
