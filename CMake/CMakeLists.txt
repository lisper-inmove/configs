cmake_minimum_required(VERSION 3.10)
project(OpenCVFromPkgConfig)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_BUILD_TYPE Debug)  # Debug 会开启调试信息，Release 会做优化

# 使用 pkg-config 获取 OpenCV 编译参数
find_package(PkgConfig REQUIRED)
pkg_check_modules(OPENCV REQUIRED opencv4)

# 为libs/*.cpp命名
file(GLOB_RECURSE native_srcs libs/*.cpp)

# 添加可执行文件和源文件
add_executable(main src/main.cpp ${native_srcs})

# 添加头文件目录（你自己的 include1 和 OpenCV 的）
# PRIVATE: 只作用于当前 target
# PUBLIC: 当前 target 和 link 到它的 target 都会使用
# INTERFACE: 只有 link 到它的 target 会使用（适用于头文件库）
target_include_directories(main PRIVATE
    ${CMAKE_SOURCE_DIR}
    ${OPENCV_INCLUDE_DIRS}
)
target_link_libraries(main ${OPENCV_LIBRARIES})

# 设置输出目录
# set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
# set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# 判断输出平台
# if (WIN32)
#     message("Building on Windows")
# elseif (UNIX)
#     message("Building on Linux/Unix")
# endif()

target_compile_options(main PRIVATE -Wall -O2)
# | 选项                   | 作用                                             |
# | -------------------- | ---------------------------------------------- |
# | `-Wall`              | 启用常见警告信息                                       |
# | `-Wextra`            | 启用额外警告                                         |
# | `-Werror`            | 把所有警告当作错误处理                                    |
# | `-O0`                | 不优化（调试用）                                       |
# | `-O2` / `-O3`        | 开启中度 / 高度优化                                    |
# | `-g`                 | 生成调试信息                                         |
# | `-march=native`      | 针对当前 CPU 优化                                    |
# | `-std=c++17`         | 使用 C++17 标准（建议使用 `set(CMAKE_CXX_STANDARD)` 管理） |
# | `-fPIC`              | 生成位置无关代码（用于构建共享库）                              |
# | `-fsanitize=address` | 启用 AddressSanitizer 内存错误检查                     |
# | `-fno-exceptions`    | 禁用异常处理支持                                       |
# | `-fno-rtti`          | 禁用 RTTI（运行时类型识别）                               |

# 添加另一个项目
# include(config/settings.cmake)

# 添加子模块
# add_subdirectory(src)
# add_subdirectory(tests)
