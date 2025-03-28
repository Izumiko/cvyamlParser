cmake_minimum_required(VERSION 3.14)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

project(cvyamlParser 
        VERSION 2.0.0
        DESCRIPTION "OpenCV YAML parser for MATLAB"
        LANGUAGES CXX)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
endif()

message(STATUS "Generated with config types: ${CMAKE_CONFIGURATION_TYPES}")
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/mex")

find_package(OpenCV REQUIRED)
message(STATUS "OpenCV library status:")
message(STATUS "    config: ${OpenCV_DIR}")
message(STATUS "    version: ${OpenCV_VERSION}")
message(STATUS "    libraries: ${OpenCV_LIBS}")
message(STATUS "    include path: ${OpenCV_INCLUDE_DIRS}")

find_package(Matlab REQUIRED MX_LIBRARY MAT_LIBRARY)

# 定义编译器特定设置的函数
function(configure_compiler_settings target)
    target_compile_options(${target} PRIVATE
        $<$<CXX_COMPILER_ID:GNU,Clang,Intel>:-Wall>
        $<$<CXX_COMPILER_ID:GNU,Clang>:-Wnon-virtual-dtor>
        $<$<AND:$<CONFIG:Debug>,$<CXX_COMPILER_ID:GNU,Clang,Intel>>:-O0 -g -fstack-protector -DDEBUG>
        $<$<AND:$<CONFIG:Release>,$<CXX_COMPILER_ID:GNU,Clang,Intel>>:-O3 -fwrapv -DNDEBUG>
    )

    target_compile_options(${target} PRIVATE
        $<$<CXX_COMPILER_ID:MSVC>:/arch:AVX2 /EHsc>
        $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Release>>:/Ox /Ot /GS- /GL /DNDEBUG>
        $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Debug>>:/Od /Zi /DDEBUG>
    )

    target_link_options(${target} PRIVATE
        $<$<AND:$<CXX_COMPILER_ID:GNU>,$<CONFIG:Release>>:-flto>
        $<$<CXX_COMPILER_ID:Clang>:-fuse-ld=lld>
        $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Release>>:/LTCG /INCREMENTAL:NO /OPT:REF>
        $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Debug>>:/DEBUG>
    )

    target_compile_definitions(${target} PRIVATE
        $<$<PLATFORM_ID:Windows>:_USE_MATH_DEFINES>
    )
endfunction()

# 创建MEX目标
if(WIN32)
    matlab_add_mex(
        NAME readcvYaml
        SRC src/readcvYaml_win.cpp
        LINK_TO ${OpenCV_LIBS})
else()
    matlab_add_mex(
        NAME readcvYaml
        SRC src/readcvYaml.cpp
        LINK_TO ${OpenCV_LIBS})
endif()
# 由于matlab_add_mex创建的目标有特殊规则，可能不能直接使用configure_compiler_settings
# 但以下是一个尝试
configure_compiler_settings(readcvYaml)

if(WIN32)
    set(OpenCV_BIN_DIR "${OpenCV_DIR}/../bin")
    if(NOT EXISTS "${OpenCV_BIN_DIR}")
        file(GLOB vc_dirs "${OpenCV_DIR}/x64/vc*")
        foreach(vc_dir ${vc_dirs})
            if(EXISTS "${vc_dir}/bin")
                set(OpenCV_BIN_DIR "${vc_dir}/bin")
                break()
            endif()
        endforeach()
    endif()
    message(STATUS "Using OpenCV_BIN_DIR: ${OpenCV_BIN_DIR}")

    string(REGEX MATCH "[0-9]+\\.[0-9]+" OpenCV_VERSION_MAJOR_MINOR ${OpenCV_VERSION})
    string(REPLACE "." "" OpenCV_VERSION_NUMBER "${OpenCV_VERSION_MAJOR_MINOR}0")

    add_custom_command(TARGET readcvYaml POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${OpenCV_BIN_DIR}/opencv_world${OpenCV_VERSION_NUMBER}$<$<CONFIG:Debug>:d>.dll
            ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/$<$<CONFIG:Debug>:Debug>$<$<CONFIG:Release>:Release>
        COMMENT "Copying OpenCV DLLs"
        VERBATIM
    )
endif()

# 创建可执行文件目标
add_executable(genyamlData generate/generate_random/genyamlData.cpp)
target_link_libraries(genyamlData PRIVATE ${OpenCV_LIBS})
configure_compiler_settings(genyamlData)

add_executable(generate_image generate/generate_image/write_test_img.cpp)
# 使用更具体的库名称而不是假设opencv_*的存在
target_link_libraries(generate_image PRIVATE ${OpenCV_LIBS})
configure_compiler_settings(generate_image)

# 为所有目标设置包含目录
foreach(target readcvYaml genyamlData generate_image)
    target_include_directories(${target} PRIVATE ${OpenCV_INCLUDE_DIRS})
endforeach()

# 启用Folders使解决方案更清晰
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# 输出一些有用的信息
message(STATUS "${PROJECT_NAME} configuration completed")
