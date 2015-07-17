cmake_minimum_required(VERSION 3.3.20150926)
project(AliasTarget)

function(getTargetsJson var)
  get_property(targets GLOBAL PROPERTY TARGETS)

  list(LENGTH targets n)
  set(i 1)

  set(json "{\n")
  set(json "${json}  \"targets\" : [\n")

  foreach(t ${targets})
    if(i LESS ${n})
      set(c ",")
    else()
      set(c "")
    endif()

    get_property(tt TARGET ${t} PROPERTY TYPE)
    set(json "${json}    { \"name\" : \"${t}\", \"type\" : \"${tt}\" }${c}\n")

    math(EXPR i "${i}+1")
  endforeach()

  set(json "${json}  ]\n")
  set(json "${json}}\n")

  set(${var} "${json}" PARENT_SCOPE)
endfunction()

function(writeTargetsJsonFile filename)
  getTargetsJson(json)
  file(WRITE "${filename}" "${json}")
endfunction()

writeTargetsJsonFile(${CMAKE_CURRENT_BINARY_DIR}/targets-0.json)

set(CMAKE_CXX_STANDARD 98)

# Those versions of the HP compiler that need a flag to get proper C++98
# template support also need a flag to use the newer C++ library.
if (CMAKE_CXX_COMPILER_ID STREQUAL HP AND
    CMAKE_CXX98_STANDARD_COMPILE_OPTION STREQUAL "+hpxstd98")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -AA")
endif ()

add_library(foo SHARED empty.cpp)
writeTargetsJsonFile(${CMAKE_CURRENT_BINARY_DIR}/targets-1.json)

add_library(PREFIX::Foo ALIAS foo)
add_library(Another::Alias ALIAS foo)

add_library(objects OBJECT object.cpp)
add_library(Alias::Objects ALIAS objects)

writeTargetsJsonFile(${CMAKE_CURRENT_BINARY_DIR}/targets-2.json)

target_compile_definitions(foo PUBLIC FOO_DEFINE)

add_library(bar SHARED empty.cpp)
target_compile_definitions(bar PUBLIC BAR_DEFINE)

target_link_libraries(foo LINK_PUBLIC $<$<STREQUAL:$<TARGET_PROPERTY:PREFIX::Foo,ALIASED_TARGET>,foo>:bar>)

add_executable(AliasTarget commandgenerator.cpp $<TARGET_OBJECTS:Alias::Objects>)
add_executable(PREFIX::AliasTarget ALIAS AliasTarget)
add_executable(Generator::Command ALIAS AliasTarget)

add_custom_command(OUTPUT commandoutput.h COMMAND Generator::Command)

add_library(bat SHARED bat.cpp "${CMAKE_CURRENT_BINARY_DIR}/commandoutput.h")
target_link_libraries(bat PREFIX::Foo)
target_include_directories(bat PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")

add_executable(targetgenerator targetgenerator.cpp)
add_executable(Generator::Target ALIAS targetgenerator)

add_custom_target(usealias Generator::Target)
add_dependencies(bat usealias)

if (NOT TARGET Another::Alias)
  message(SEND_ERROR "Another::Alias is not considered a target.")
endif()

get_target_property(_alt PREFIX::Foo ALIASED_TARGET)
if (NOT ${_alt} STREQUAL foo)
  message(SEND_ERROR "ALIASED_TARGET is not foo: ${_alt}")
endif()

get_property(_alt2 TARGET PREFIX::Foo PROPERTY ALIASED_TARGET)
if (NOT ${_alt2} STREQUAL foo)
  message(SEND_ERROR "ALIASED_TARGET is not foo.")
endif()

add_library(iface INTERFACE)
add_library(Alias::Iface ALIAS iface)

get_target_property(_notAlias1 foo ALIASED_TARGET)
if (NOT DEFINED _notAlias1)
  message(SEND_ERROR "_notAlias1 is not defined")
endif()
if (_notAlias1)
  message(SEND_ERROR "_notAlias1 is defined, but foo is not an ALIAS")
endif()
if (NOT _notAlias1 STREQUAL _notAlias1-NOTFOUND)
  message(SEND_ERROR "_notAlias1 not defined to a -NOTFOUND variant")
endif()

get_property(_notAlias2 TARGET foo PROPERTY ALIASED_TARGET)
if (NOT DEFINED _notAlias2)
  message(SEND_ERROR "_notAlias2 is not defined")
endif()
if (_notAlias2)
  message(SEND_ERROR "_notAlias2 is defined, but foo is not an ALIAS")
endif()
if (NOT _notAlias2 STREQUAL _notAlias2-NOTFOUND)
  message(SEND_ERROR "_notAlias2 not defined to a -NOTFOUND variant")
endif()

writeTargetsJsonFile(${CMAKE_CURRENT_BINARY_DIR}/targets-3.json)
