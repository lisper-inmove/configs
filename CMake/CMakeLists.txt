cmake_minimum_required(VERSION 3.10)
project(OpenCVFromPkgConfig)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_BUILD_TYPE Debug)

# 使用 pkg-config 获取 OpenCV 编译参数
find_package(PkgConfig REQUIRED)
pkg_check_modules(OPENCV REQUIRED opencv4)

# 为libs/*.cpp命名
file(GLOB_RECURSE native_srcs libs/*.cpp)

# 添加可执行文件和源文件
add_executable(main src/main.cpp ${native_srcs})

# 添加头文件目录（你自己的 include1 和 OpenCV 的）
target_include_directories(main PRIVATE
    ${CMAKE_SOURCE_DIR}
    ${OPENCV_INCLUDE_DIRS}
)

# 链接 OpenCV 库
target_link_libraries(main ${OPENCV_LIBRARIES})
