include(Lint.cmake)

function(PROTOBUF_GENERATE_CPP SRCS HDRS)
  if(NOT ARGN)
    message(SEND_ERROR "Error: PROTOBUF_GENERATE_CPP() called without any proto files")
    return()
  endif()

  if(PROTOBUF_GENERATE_CPP_APPEND_PATH)
    # Create an include path for each file specified
    foreach(FIL ${ARGN})
      get_filename_component(ABS_FIL ${FIL} ABSOLUTE)
      get_filename_component(ABS_PATH ${ABS_FIL} PATH)
      list(FIND _protobuf_include_path ${ABS_PATH} _contains_already)
      if(${_contains_already} EQUAL -1)
          list(APPEND _protobuf_include_path -I ${ABS_PATH})
      endif()
    endforeach()
  else()
    set(_protobuf_include_path -I ${CMAKE_CURRENT_SOURCE_DIR})
  endif()

  if(DEFINED Protobuf_IMPORT_DIRS)
    foreach(DIR ${Protobuf_IMPORT_DIRS})
      get_filename_component(ABS_PATH ${DIR} ABSOLUTE)
      list(FIND _protobuf_include_path ${ABS_PATH} _contains_already)
      if(${_contains_already} EQUAL -1)
          list(APPEND _protobuf_include_path -I ${ABS_PATH})
      endif()
    endforeach()
  endif()

  set(${SRCS})
  set(${HDRS})
  foreach(FIL ${ARGN})
    get_filename_component(ABS_FIL ${FIL} ABSOLUTE)
    get_filename_component(FIL_WE ${FIL} NAME_WE)

    list(APPEND ${SRCS} "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.cc")
    list(APPEND ${HDRS} "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.h")

    add_custom_command(
      OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.cc"
             "${CMAKE_CURRENT_BINARY_DIR}/${FIL_WE}.pb.h"
      COMMAND  ${Protobuf_PROTOC_EXECUTABLE}
      ARGS --cpp_out  ${CMAKE_CURRENT_BINARY_DIR} ${_protobuf_include_path} ${ABS_FIL}
      DEPENDS ${ABS_FIL} ${Protobuf_PROTOC_EXECUTABLE}
      COMMENT "Running C++ protocol buffer compiler on ${FIL}"
      VERBATIM )
  endforeach()

  set_source_files_properties(${${SRCS}} ${${HDRS}} PROPERTIES GENERATED TRUE)
  set(${SRCS} ${${SRCS}} PARENT_SCOPE)
  set(${HDRS} ${${HDRS}} PARENT_SCOPE)
endfunction()

if (WIN32)
set (Protobuf_PROTOC_EXECUTABLE protoc)
else()
set (Protobuf_PROTOC_EXECUTABLE ${CMAKE_BINARY_DIR}/protoc)
set (PROTOBUF_PROTOC_EXECUTABLE ${CMAKE_BINARY_DIR}/protoc)
endif()

PROTOBUF_GENERATE_CPP(ProtoSources ProtoHeaders
  playback.proto
  )

include_directories (
  ${CMAKE_SOURCE_DIR}/retrace
  ${CMAKE_SOURCE_DIR}/lib/image
  ${CMAKE_BINARY_DIR}/dispatch
  ${CMAKE_SOURCE_DIR}/dispatch
  ${CMAKE_SOURCE_DIR}/common
  ${CMAKE_SOURCE_DIR}/helpers
  ${CMAKE_BINARY_DIR}/retrace/daemon
  ${CMAKE_SOURCE_DIR}/retrace/daemon/gldispatch
  ${CMAKE_SOURCE_DIR}/thirdparty/md5
  )

set (RETRACE_LINUX_SOURCE
  glframe_batch.cpp
  glframe_gpu_speed_linux.cpp
  glframe_os_linux.cpp
  glframe_perf_enabled_linux.cpp
  glframe_stderr.cpp
  glframe_thread_linux.cpp
  )

set (RETRACE_WIN_SOURCE
  glframe_os_win.cpp
  glframe_thread_win.cpp
  glframe_win_shaders.cpp
  )

if (WIN32)
  set (RETRACE_OS_SOURCE ${RETRACE_WIN_SOURCE})
else()
  set (RETRACE_OS_SOURCE ${RETRACE_LINUX_SOURCE})
endif()

set (RETRACE_SOURCES
  ${RETRACE_OS_SOURCE}
  glframe_batch.hpp
  glframe_gpu_speed.hpp
  glframe_metrics.cpp
  glframe_metrics.hpp
  glframe_os.hpp
  glframe_logger.cpp
  glframe_logger.hpp
  glframe_perf_enabled.hpp
  glframe_retrace.cpp
  glframe_retrace.hpp
  glframe_retrace_context.cpp
  glframe_retrace_context.hpp
  glframe_retrace_interface.hpp
  glframe_retrace_render.cpp
  glframe_retrace_render.hpp
  glframe_retrace_skeleton.cpp
  glframe_retrace_skeleton.hpp
  glframe_retrace_stub.cpp
  glframe_retrace_stub.hpp
  glframe_socket.hpp
  glframe_socket.cpp
  glframe_state.cpp
  glframe_state.hpp
  glframe_stderr.hpp
  glframe_thread.hpp
  glframe_traits.hpp
  glframe_uniforms.cpp
  glframe_uniforms.hpp
  glretrace_globals.cpp
  )  

add_library(retrace_daemon STATIC
  ${ProtoHeaders}
  ${ProtoSources}
  ${RETRACE_SOURCES}
  )

target_link_libraries(retrace_daemon
  glretrace_common
  retrace_gldispatch
  libprotobuf
  )

add_dependencies(retrace_daemon protoc)

if (NOT WIN32)
  Lint( RETRACE_SOURCES )
endif()

add_subdirectory (framestat)
if (ENABLE_WAFFLE AND Qt5Widgets_FOUND)
  add_subdirectory (test)
  add_subdirectory (ui)
  add_subdirectory (bargraph)
endif ()
add_subdirectory (server)
add_subdirectory (gldispatch)
