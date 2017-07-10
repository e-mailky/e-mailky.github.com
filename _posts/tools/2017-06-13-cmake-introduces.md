```
cmake_minimum_required(VERSION 2.8)

# 包含其他cmake文件
# 通用配置选项，和通用的全局函数
include(build-windows/common.cmake)

#项目
project(webrtc)

#比较有用的内置变量 当前目录
set(current_dir ${CMAKE_CURRENT_SOURCE_DIR})

# 判断目录或者文件是否存在
set (directx_sdk_path "${webrtc_root}/third_party/directxsdk/files")
if (NOT EXISTS ${directx_sdk_path})
	message(FATAL_ERROR "The directx_sdk_path(${directx_sdk_path}) doesn't exist.")
endif()

#计算文件名的父目录
get_filename_component(webrtc_root ${CMAKE_SOURCE_DIR} DIRECTORY)
#打印一条提示信息
message (STATUS "webrtc_root = ${webrtc_root}")

# 添加全局的include 目录，所有 子目录都会引用这个头文件目录
# 可以采用相对目录， 这个是追加到include'dir里面去的
include_directories("${webrtc_root}")
include_directories("interface"
					"${webrtc_root}/webrtc"
					"../codecs/cng/include")

#全局的编译选项，所有子目录的项目都包含这个选项
# 所有的lib和exe统一使用static linking
add_compile_options("/MT")

#全局的预定义宏，应用到所有的子目录
add_definitions(-DWEBRTC_WIN -DNOMINMAX)

# 包含子目录，继承当前的 include dir和 编译选项等设置
add_subdirectory(common_audio)

# 如果这个“子目录”是另外父目录里面的，需要指定第二个参数
add_subdirectory("${webrtc_root}/third_party/gflags" gflags)

set (NetEq4_source
	"interface/audio_decoder.h"
	"interface/neteq.h"
	"accelerate.cc"
	"accelerate.h"
	"audio_decoder_impl.cc"
	"audio_decoder_impl.h"
	"audio_decoder.cc"
	"audio_multi_vector.cc"
	"audio_multi_vector.h"
	"audio_vector.cc"
	"audio_vector.h"
	"background_noise.cc"
	"background_noise.h")
# 指定一个target 动态链接库和他的源码，
# 创建可执行文件的target用 add_executable命令
add_library(NetEq4 STATIC ${NetEq4_source})
#指定依赖的链接库，依赖管理自动继承，
#比如下面这个表示编译NetEq4的时候，需要链接到G711
# 那么如果另外一个target依赖NetEq4 的话，cmake也会
# 自动配置他也依赖 G711了
# add_dependencies 指定构建顺序，但没有linker的库关联
target_link_libraries(NetEq4 G711)
target_link_libraries(NetEq4 G722)

# 下面这几个命令为某个target指定编译选项和头文件包含目录
# 相比较前面的include_directories等全局的命令，这个的优点
# 是可以针对某个具体的target来设置
# 不要使用 set_target_properties 命令来设置编译选项之类的，
# 因为set_target_properties 不是追加方式的，只能设置一次，可能
# 会覆盖之前的设置
# set_source_files_properties 针对某个文件设置编译选项，某些情况下
# 有点用处 吧
# 注意下面 的PRIVATE 属性和 PUBLIC的区别，如果是 PUBLIC的话，这个
# 属性也会那些依赖连接库继承过去。PRIVATE就是只会对自己有效。
# 比如某个包含目录的设置，一般作为接口的头文件库使用的就可以使用PUBLIC
# 了，这样如果使用target_link_libraries设置了依赖之后，就不用再设置这个
# include dir了。
target_compile_definitions(audio_decoder_unittests
	PRIVATE "AUDIO_DECODER_UNITTEST"
	PRIVATE "WEBRTC_CODEC_G722"
	PRIVATE "WEBRTC_CODEC_ILBC"
	PRIVATE "WEBRTC_CODEC_ISACFX"
	PRIVATE "WEBRTC_CODEC_ISAC"
	PRIVATE "WEBRTC_CODEC_PCM16")
target_include_directories(audio_decoder_unittests
	PRIVATE "interface"
	PRIVATE "test"
	PRIVATE "../codecs/isac/main/interface"
	PRIVATE "${webrtc_root}/third_party/testing/gtest/include"
	PRIVATE "${webrtc_root}/third_party/testing/gmock/include"
	PRIVATE  "../codecs/ilbc/interface"
	PRIVATE  "../codecs/pcm16b/include"
	PRIVATE "../codecs/g722/include"
	PRIVATE "../codecs/g711/include")
target_compile_options(RTPencode PRIVATE "/wd4267")

# 下面这个使用PUBLIC，希望依赖gflags的库，就自动包含这些宏的定义和头文件目录
# These macros exist so flags and symbols are properly
# exported when building DLLs. Since we don't build DLLs, we
# need to disable them.
target_compile_definitions(gflags
	PUBLIC "GFLAGS_DLL_DECL="
	PUBLIC "GFLAGS_DLL_DECLARE_FLAG="
	PUBLIC "GFLAGS_DLL_DEFINE_FLAG=")
if (MSVC)
	target_compile_options(gflags PRIVATE "/wd4005" PRIVATE "/wd4267")
	target_include_directories(gflags
		PUBLIC "src/windows"
		PUBLIC "src/windows/gflags"
		PUBLIC "src")
else(UNIX)
	target_include_directories(gflags
		PUBLIC "src/"
		PUBLIC "src/gflags")
endif()

#导入外部的库作为链接时的target使用。下面用add_library命令为外部生成的libfoo.a
#创建一个叫做foo的target，便于后面target_link_libraries的链接引用。
#IMPORTED_LOCATION属性执行库的路径，也可以设置外部的动态链接的库。
#例子来自 CMake/Tutorials/Exporting and Importing Targets
#http://www.cmake.org/Wiki/CMake/Tutorials/Exporting_and_Importing_Targets

 add_library(foo STATIC IMPORTED)
 set_property(TARGET foo PROPERTY IMPORTED_LOCATION /path/to/libfoo.a)
 add_executable(myexe src1.c src2.c)
 target_link_libraries(myexe foo)

 add_library(bar SHARED IMPORTED)
 set_property(TARGET bar PROPERTY IMPORTED_LOCATION c:/path/to/bar.dll)
 set_property(TARGET bar PROPERTY IMPORTED_IMPLIB c:/path/to/bar.lib)
 add_executable(myexe src1.c src2.c)
 target_link_libraries(myexe bar)

 add_library(foo STATIC IMPORTED)
 set_property(TARGET foo PROPERTY IMPORTED_LOCATION_RELEASE c:/path/to/foo.lib)
 set_property(TARGET foo PROPERTY IMPORTED_LOCATION_DEBUG   c:/path/to/foo_d.lib)
 add_executable(myexe src1.c src2.c)
 target_link_libraries(myexe foo)

#如果需要把imported的library导出到其他目录供使用，需要在imported好哦买加上
#GLOBAL，表示全局。这样不在同一个子目录下的cmakefile里面也可以引用这个library。
#这样
 add_library(foo STATIC IMPORTED  GLOBAL)

#指定链接选项
#编译选择可以用 target_compile_options  add_compile_options add_definitions
#等来做，但链接选项的话，最新的cmake 2.8.12都还没有一个类似target_compile_options的
#命令，只能设置全局的cmake 变量。只能用set_target_properties命令来设置了，
#但这个为了不覆盖之前的参数，只能先set_target_properties获取旧的值，然后再追加设置了
#。http://binglongx.wordpress.com/2013/06/30/add-arbitrary-compilelink-flags-in-cmake/
#这里把它封装陈一个宏比较方便使用
############################################################################################################
# AddCompileLinkFlags.cmake
############################################################################################################

############################################################################################################
# Append str to a string property of a target.
# target:      string: target name.
# property:            name of target’s property. e.g: COMPILE_FLAGS, or LINK_FLAGS
# str:         string: string to be appended to the property
macro(my_append_target_property target property str)
  get_target_property(current_property ${target} ${property})
  if(NOT current_property) # property non-existent or empty
      set_target_properties(${target} PROPERTIES ${property} ${str})
  else()
      set_target_properties(${target} PROPERTIES ${property} "${current_property} ${str}")
  endif()
endmacro(my_append_target_property)

############################################################################################################
# Add/append compile flags to a target.
# target: string: target name.
# flags : string: compile flags to be appended
macro(my_add_compile_flags target flags)
  my_append_target_property(${target} COMPILE_FLAGS ${flags})
endmacro(my_add_compile_flags)

############################################################################################################
# Add/append link flags to a target.
# target: string: target name.
# flags : string: link flags to be appended
macro(my_add_link_flags target flags)
  my_append_target_property(${target} LINK_FLAGS ${flags})
endmacro(my_add_link_flags)

#Now, to turn off Image Has Safe Exception Handlers linker option of my target, I just need to use my_add_link_flags:
include(AddCompileLinkFlags.cmake)
add_executable(test_proj test.cpp ${ConfigFiles} ${GeneratedFiles})  # VS project
my_add_link_flags(test_proj "/SAFESEH:NO")

#不过这个add_compile_flags是没必要的了，新的cmake 2.8.12版已经有target_compile_options

#安装程序
#这个命令可以配合cpack，自动生成debian安装包等。

#安装外部已经编译好的程序
	install(PROGRAMS
			  ${PocoLibPathPrefix}/libPocoFoundation.so
			  ${PocoLibPathPrefix}/libPocoXML.so
			  ${PocoLibPathPrefix}/libPocoNet.so
			  ${PocoLibPathPrefix}/libPocoUtil.so
			  ${PocoLibPathPrefix}/libPocoFoundation.so.16
			  ${PocoLibPathPrefix}/libPocoXML.so.16
			  ${PocoLibPathPrefix}/libPocoNet.so.16
			  ${PocoLibPathPrefix}/libPocoUtil.so.16
			DESTINATION ${ILIB_INSTALL_SUFFIX})

#   安装自己项目内部编译生成的的target，
         install(TARGETS SMPP LIBRARY DESTINATION ${ILIB_INSTALL_SUFFIX})

#按引用传递参数？或者说修改参数的原始值/修改父级变量的值。
#注意set(${source_list} ${new_source_list} PARENT_SCOPE)  一句的用法。
#list的名字和值的写法 ${${source_list}}
# 把符合正则表达式的源文件名从列表中排除出去
function (exclude_regex_matched_source regex source_list)
	set (new_source_list)
	foreach (source_name ${${source_list}})
		if (NOT source_name MATCHES regex)
			list(APPEND new_source_list ${source_name})
		endif()
	endforeach()
	set(${source_list} ${new_source_list} PARENT_SCOPE)
endfunction()

#外部自定义编译命令，比如golang程序
if (MSVC)
  set (TEST_CDR_BIN ${CMAKE_CURRENT_SOURCE_DIR}/test_cdr.exe)
  set (BUILD_GOLANG_CMD build_test_cdr.bat)
elseif (UNIX)
  set (TEST_CDR_BIN ${CMAKE_CURRENT_SOURCE_DIR}/test_cdr)
  set (BUILD_GOLANG_CMD ${CMAKE_CURRENT_SOURCE_DIR}/build_test_cdr.sh)
endif()

# BUILD_GOLANG_CMD 命令生成${TEST_CDR_BIN} 文件。
file(REMOVE ${TEST_CDR_BIN})
add_custom_command(OUTPUT ${TEST_CDR_BIN}
                   COMMAND ${BUILD_GOLANG_CMD}
                   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                   )

add_custom_target(smsc_cdr_build_golang_program ALL
                  DEPENDS ${TEST_CDR_BIN}
                  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                  COMMENT "build smsc_cdr"
                  )
install(PROGRAMS ${TEST_CDR_BIN} DESTINATION ${TEST_BIN_INSTALL_SUFFIX})
```
