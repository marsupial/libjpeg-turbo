if(NOT DEFINED NASM)
  find_program(NASM NAMES nasm yasm DOC "Path to NASM/YASM executable")
endif()
message(STATUS "NASM = ${NASM}")

if(SIMD_X86_64)
  set(NAFLAGS -D__x86_64__)
  if(WIN32)
    set(NAFLAGS -fwin64 -DWIN64 ${NAFLAGS} -I${CMAKE_SOURCE_DIR}/win)
  elseif(APPLE)
    set(NAFLAGS -fmacho64 -D__MACHO__ ${NAFLAGS})
  elseif("${CMAKE_SYSTEM}" MATCHES "Linux")
    set(NAFLAGS -felf64 -DELF ${NAFLAGS})
  else()
    message(FATAL_ERROR "Unkown object 64 format")
  endif()
else()
  if(BORLAND)
    set(NAFLAGS -fobj -DOBJ32)
  elseif(WIN32)
    set(NAFLAGS -fwin32 -DWIN32 -I${CMAKE_SOURCE_DIR}/win)
  elseif(APPLE)
    set(NAFLAGS -fmacho32 -D__MACHO__)
  elseif("${CMAKE_SYSTEM}" MATCHES "Linux")
    set(NAFLAGS -felf32 -DELF)
  else()
    message(FATAL_ERROR "Unkown object 32 format")
  endif()
endif()
set(NAFLAGS ${NAFLAGS} -I${CMAKE_CURRENT_SOURCE_DIR}/)
if(NOT WIN32)
  set(NAFLAGS ${NAFLAGS} -I${CMAKE_CURRENT_BINARY_DIR}/)
endif()

# This only works if building from the command line.  There is currently no way
# to set a variable's value based on the build type when using the MSVC IDE.
if(CMAKE_BUILD_TYPE STREQUAL "Debug"
  OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
  set(NAFLAGS ${NAFLAGS} -g)
endif()

if(SIMD_X86_64)
  set(SIMD_BASENAMES jfdctflt-sse-64 jccolor-sse2-64 jcgray-sse2-64
    jchuff-sse2-64 jcsample-sse2-64 jdcolor-sse2-64 jdmerge-sse2-64
    jdsample-sse2-64 jfdctfst-sse2-64 jfdctint-sse2-64 jidctflt-sse2-64
    jidctfst-sse2-64 jidctint-sse2-64 jidctred-sse2-64 jquantf-sse2-64
    jquanti-sse2-64)
  message(STATUS "Building x86_64 SIMD extensions")
else()
  set(SIMD_BASENAMES jsimdcpu jfdctflt-3dn jidctflt-3dn jquant-3dn jccolor-mmx
    jcgray-mmx jcsample-mmx jdcolor-mmx jdmerge-mmx jdsample-mmx jfdctfst-mmx
    jfdctint-mmx jidctfst-mmx jidctint-mmx jidctred-mmx jquant-mmx jfdctflt-sse
    jidctflt-sse jquant-sse jccolor-sse2 jcgray-sse2 jchuff-sse2 jcsample-sse2
    jdcolor-sse2 jdmerge-sse2 jdsample-sse2 jfdctfst-sse2 jfdctint-sse2
    jidctflt-sse2 jidctfst-sse2 jidctint-sse2 jidctred-sse2 jquantf-sse2
    jquanti-sse2)
  message(STATUS "Building i386 SIMD extensions")
endif()

if(MSVC_IDE)
  set(OBJDIR "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CFG_INTDIR}")
else()
  set(OBJDIR ${CMAKE_CURRENT_BINARY_DIR})
endif()

file(GLOB INC_FILES *.inc)

foreach(file ${SIMD_BASENAMES})
  set(DEPFILE "")
  set(SIMD_SRC ${CMAKE_CURRENT_SOURCE_DIR}/${file}.asm)
  if(${file} MATCHES jccolor)
    set(DEPFILE ${file})
    string(REGEX REPLACE "jccolor" "jccolext" DEPFILE ${DEPFILE})
    set(DEPFILE ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE}.asm)
  endif()
  if(${file} MATCHES jcgray)
    set(DEPFILE ${file})
    string(REGEX REPLACE "jcgray" "jcgryext" DEPFILE ${DEPFILE})
    set(DEPFILE ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE}.asm)
  endif()
  if(${file} MATCHES jdcolor)
    set(DEPFILE ${file})
    string(REGEX REPLACE "jdcolor" "jdcolext" DEPFILE ${DEPFILE})
    set(DEPFILE ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE}.asm)
  endif()
  if(${file} MATCHES jdmerge)
    set(DEPFILE ${file})
    string(REGEX REPLACE "jdmerge" "jdmrgext" DEPFILE ${DEPFILE})
    set(DEPFILE ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE}.asm)
  endif()
  set(SIMD_OBJ ${OBJDIR}/${file}.obj)
  add_custom_command(OUTPUT ${SIMD_OBJ}
    DEPENDS ${SIMD_SRC} ${DEPFILE} ${INC_FILES}
    COMMAND ${NASM} ${NAFLAGS} ${SIMD_SRC} -o${SIMD_OBJ})
  set(SIMD_OBJS ${SIMD_OBJS} ${SIMD_OBJ})
endforeach()

set(SIMD_OBJS ${SIMD_OBJS} PARENT_SCOPE)
add_custom_target(simd DEPENDS ${SIMD_OBJS})

if (NOT WIN32)
  get_filename_component(ppcompiler ${CMAKE_C_COMPILER} NAME)
  if(ppcompiler STREQUAL "ccache" OR ppcompiler STREQUAL "distcc")
    separate_arguments(pp_arglist UNIX_COMMAND "${CMAKE_C_COMPILER_ARG1}")
    list(GET pp_arglist 0 ppcompiler)
  endif()
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/jsimdcfg.inc
    COMMAND "${ppcompiler}" -E -I../ -I. "${CMAKE_CURRENT_SOURCE_DIR}/jsimdcfg.inc.h" | grep \"^[;%]\" | sed 's/_cpp_protection_//' | sed 's/% define/%define/g' > ${CMAKE_CURRENT_BINARY_DIR}/jsimdcfg.inc
    MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/jsimdcfg.inc.h
    COMMENT "Preprocessing jsimdcfg.inc.h")
  add_custom_target(jsimdcfg.inc DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/jsimdcfg.inc
                                         ${CMAKE_BINARY_DIR}/jconfig.h
                                         ${CMAKE_BINARY_DIR}/jconfigint.h)
  add_dependencies(simd jsimdcfg.inc)
endif()

