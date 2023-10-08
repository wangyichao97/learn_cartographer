# Copyright 2016 The Cartographer Authors
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 2.8.12)  # Ships with Ubuntu 14.04 (Trusty)

project(cartographer_ros)

#定义变量，内容为ros包的依赖项
set(PACKAGE_DEPENDENCIES
  cartographer_ros_msgs
  geometry_msgs
  message_runtime
  nav_msgs
  pcl_conversions
  rosbag
  roscpp
  roslib
  sensor_msgs
  std_msgs
  tf2
  tf2_eigen
  tf2_ros
  urdf
  visualization_msgs
)

#检查操作系统是否为Windows
if(WIN32)
  set(Boost_USE_STATIC_LIBS FALSE)
endif()
# For yet unknown reason, if Boost is find_packaged() after find_package(cartographer),
# some Boost libraries including Thread are set to static, despite Boost_USE_STATIC_LIBS,
# which causes linking problems on windows due to shared/static Thread clashing.
# PCL also finds Boost. Work around by moving before find_package(cartographer).

#找到所需要的包，需要最先找到Boost包，求至少找到Boost库的system和iostreams两个组件，REQUIRED参数表示这些组件是必需的。
find_package(Boost REQUIRED COMPONENTS system iostreams)
#用于查找和配置点云库（PCL）的依赖项。它要求至少找到PCL库的common组件
find_package(PCL REQUIRED COMPONENTS common)

find_package(cartographer REQUIRED)

include("${CARTOGRAPHER_CMAKE_DIR}/functions.cmake")
#定义了一个CMake选项，允许您选择是否构建需要Cartographer gRPC支持的功能。默认情况下为false，即不构建这些功能。
option(BUILD_GRPC "build features that require Cartographer gRPC support" false)

#初始化Cartographer项目，以便进行构建。
google_initialize_cartographer_project()
#启用Cartographer项目的测试
google_enable_testing()

set(CARTOGRAPHER_GMOCK_LIBRARIES ${GMOCK_LIBRARIES})

find_package(catkin REQUIRED COMPONENTS ${PACKAGE_DEPENDENCIES})

include(FindPkgConfig)

find_package(absl REQUIRED)
find_package(LuaGoogle REQUIRED)
find_package(Eigen3 REQUIRED)

#对urdfdom_headers库的版本进行检查，并根据版本号执行不同的操作
find_package(urdfdom_headers REQUIRED)
if(DEFINED urdfdom_headers_VERSION)
  if(${urdfdom_headers_VERSION} GREATER 0.4.1)
    add_definitions(-DURDFDOM_HEADERS_HAS_SHARED_PTR_DEFS)
  endif()
endif()

include_directories(
  ${urdfdom_headers_INCLUDE_DIRS}
)

# Override Catkin's GTest configuration to use GMock.
set(GTEST_FOUND TRUE)
set(GTEST_INCLUDE_DIRS ${GMOCK_INCLUDE_DIRS})
set(GTEST_LIBRARIES ${CARTOGRAPHER_GMOCK_LIBRARIES})

catkin_package(
  CATKIN_DEPENDS            #ros内的依赖
    ${PACKAGE_DEPENDENCIES}
  DEPENDS                   #ros外的依赖
    # TODO(damonkohler): This should be here but causes Catkin to abort because
    # protobuf specifies a library '-lpthread' instead of just 'pthread'.
    # CARTOGRAPHER
    PCL
    EIGEN3
    Boost
    urdfdom_headers
    # 包含了头文件目录的相对路径，通常是"."，表示包含当前目录
  INCLUDE_DIRS "."

    # 包含了此ROS包生成的库的名称，${PROJECT_NAME}是ROS包的名称，用于表示将要生成的库。
  LIBRARIES ${PROJECT_NAME}
)

# 将cartographer_ros目录中的所有以.cc和.h为扩展名的文件（包括子目录中的文件）存储在ALL_SRCS变量中
file(GLOB_RECURSE ALL_SRCS "cartographer_ros/*.cc" "cartographer_ros/*.h")
# 将所有以_test.cc为扩展名的测试文件（包括子目录中的文件）存储在ALL_TESTS变量中
file(GLOB_RECURSE ALL_TESTS "cartographer_ros/*_test.cc")
# 将所有以_main.cc为扩展名的可执行文件（包括子目录中的文件）存储在ALL_EXECUTABLES变量中
file(GLOB_RECURSE ALL_EXECUTABLES "cartographer_ros/*_main.cc")
# 将所有在cartographer_ros/cartographer_grpc目录中的文件存储在ALL_GRPC_FILES变量中
file(GLOB_RECURSE ALL_GRPC_FILES "cartographer_ros/cartographer_grpc/*")

#这些行命令从ALL_SRCS变量中移除ALL_TESTS和ALL_EXECUTABLES文件，以确保这些文件不会被包含在编译中。
list(REMOVE_ITEM ALL_SRCS ${ALL_TESTS})
list(REMOVE_ITEM ALL_SRCS ${ALL_EXECUTABLES})
if (NOT ${BUILD_GRPC})
  list(REMOVE_ITEM ALL_SRCS ${ALL_GRPC_FILES})
  list(REMOVE_ITEM ALL_TESTS ${ALL_GRPC_FILES})
  list(REMOVE_ITEM ALL_EXECUTABLES ${ALL_GRPC_FILES})
endif()
# 创建一个名为${PROJECT_NAME}的静态库，并将ALL_SRCS中列出的所有源文件添加到库中。
add_library(${PROJECT_NAME} STATIC ${ALL_SRCS})
#进入cartographer_ros目录并执行其中的CMakeLists.txt文件，生成可执行文件
add_subdirectory("cartographer_ros")

target_link_libraries(${PROJECT_NAME} PUBLIC cartographer)

# Lua
# 包含Lua头文件的目录
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC ${LUA_INCLUDE_DIR})

# PCL
# 包含PCL头文件的目录
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC ${PCL_INCLUDE_DIRS})
# 使用target_link_libraries函数将项目与${PCL_LIBRARIES}链接，确保项目能够使用PCL库
target_link_libraries(${PROJECT_NAME} PUBLIC ${PCL_LIBRARIES})
# 设置了一个字符串变量，其中包含了PCL定义的黑名单。这些定义可能包含与特定CPU架构相关的编译选项
set(BLACKLISTED_PCL_DEFINITIONS " -march=native -msse4.2 -mfpmath=sse ")
# 循环遍历${PCL_DEFINITIONS}列表中的每个定义
foreach(DEFINITION ${PCL_DEFINITIONS})
  # list（）用于查找${DEFINITION}是否存在于PCL定义的黑名单中，返回值DEFINITIONS_INDEX是查找结果的索引，如果没有找到，则为-1。
  list (FIND BLACKLISTED_PCL_DEFINITIONS "${DEFINITION}" DEFINITIONS_INDEX)
  # 如果${DEFINITION}在黑名单中则continue()语句将跳过当前循环迭代，不会将黑名单中的定义添加到后续的编译标志中。
  if (${DEFINITIONS_INDEX} GREATER -1)
    continue()
  endif()
  # 如果${DEFINITION}不在黑名单中，将被添加到TARGET_COMPILE_FLAGS中，并将这些定义添加到项目的编译选项中
  set(TARGET_COMPILE_FLAGS "${TARGET_COMPILE_FLAGS} ${DEFINITION}")
endforeach()

# Eigen
# 包含Eigen头文件的目录
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC
  "${EIGEN3_INCLUDE_DIR}")

# Boost
# 包含 Boost头文件的目录并链接到项目里
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC
  "${Boost_INCLUDE_DIRS}")
target_link_libraries(${PROJECT_NAME} PUBLIC ${Boost_LIBRARIES})

# Catkin
# 包含 Catkin头文件的目录并链接到项目里，并添加依赖关系
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC ${catkin_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} PUBLIC ${catkin_LIBRARIES})
add_dependencies(${PROJECT_NAME} ${catkin_EXPORTED_TARGETS})

# Add the binary directory first, so that port.h is included after it has
# been generated.
# 使用target_include_directories函数将以下目录添加到项目的包含目录中：
target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}>  # 在项目构建期间CMake生成的中间文件和可执行文件目录。
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>  # 项目源代码的目录。
    $<INSTALL_INTERFACE:include>              # 安装目录下的include目录。
)

# 设置一些编译选项
set(TARGET_COMPILE_FLAGS "${TARGET_COMPILE_FLAGS} ${GOOG_CXX_FLAGS}")
set_target_properties(${PROJECT_NAME} PROPERTIES
  COMPILE_FLAGS ${TARGET_COMPILE_FLAGS})

# 用于配置和构建测试相关的内容
if (CATKIN_ENABLE_TESTING)
  # 遍历${ALL_TESTS}列表中的每个测试源文件。
  foreach(TEST_SOURCE_FILENAME ${ALL_TESTS})
    # 根据${TEST_SOURCE_FILENAME}获取测试的名称（不包括扩展名），并将其存储在变量TEST_NAME中。
    get_filename_component(TEST_NAME ${TEST_SOURCE_FILENAME} NAME_WE)
    # 添加一个gtest测试。
    catkin_add_gtest(${TEST_NAME} ${TEST_SOURCE_FILENAME})
    # catkin_add_gtest uses a plain (i.e. no PUBLIC/PRIVATE/INTERFACE) call to
    # target_link_libraries. That forces us to do the same.
    # 这行代码用于将gtest和gmock库链接到${TEST_NAME}目标中。这些库通常用于编写和运行测试。
    target_link_libraries(${TEST_NAME} ${GMOCK_LIBRARIES} ${GTEST_MAIN_LIBRARIES})
    # 将${LUA_INCLUDE_DIR}添加到${TEST_NAME}目标的包含目录中，确保测试可以访问Lua的头文件。
    target_include_directories(${TEST_NAME} SYSTEM PUBLIC ${LUA_INCLUDE_DIR})
    target_link_libraries(${TEST_NAME} ${LUA_LIBRARIES})

    target_include_directories(${TEST_NAME} SYSTEM PUBLIC ${catkin_INCLUDE_DIRS})
    target_link_libraries(${TEST_NAME} ${catkin_LIBRARIES})
    add_dependencies(${TEST_NAME} ${catkin_EXPORTED_TARGETS})

    target_link_libraries(${TEST_NAME} cartographer)
    target_link_libraries(${TEST_NAME} ${PROJECT_NAME})

    #编译相关
    set_target_properties(${TEST_NAME} PROPERTIES COMPILE_FLAGS ${TARGET_COMPILE_FLAGS})
    # Needed for dynamically linked GTest on Windows.
    # 查是否在Windows操作系统上进行编译。
    if (WIN32)
      target_compile_definitions(${TEST_NAME} PUBLIC -DGTEST_LINKED_AS_SHARED_LIBRARY)
    endif()
  endforeach()
endif()

# 用于配置项目的安装规则，以确定在构建和安装项目时如何分发文件和目标
# 将以下目录中的内容安装到${CATKIN_PACKAGE_SHARE_DESTINATION}目录下：
install(DIRECTORY launch urdf configuration_files   # 目录的内容
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}   # share/${PROJECT_NAME}目录中
)

#将scripts/tf_remove_frames.py文件安装到${CATKIN_PACKAGE_BIN_DESTINATION}目录中
install(PROGRAMS scripts/tf_remove_frames.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

#将项目中的${PROJECT_NAME}目标安装到不同的目标目录中，具体取决于目标的类型：
install(TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION}
)

# Install source headers.
# 用于将项目中的头文件（.h文件）安装到指定的目标位置
# 使用file命令和GLOB_RECURSE选项来查找项目中所有以.h为扩展名的头文件
file(GLOB_RECURSE HDRS "cartographer_ros/*.h")
# 遍历找到的头文件列表
foreach(HDR ${HDRS})
  # 计算每个头文件相对于${PROJECT_SOURCE_DIR}的相对路径，并将结果存储在REL_FIL变量中。
  file(RELATIVE_PATH REL_FIL ${PROJECT_SOURCE_DIR} ${HDR})
  # 头文件的相对路径中提取的目录部分，即将相对路径中的文件名去掉，只保留所在路径
  get_filename_component(INSTALL_DIR ${REL_FIL} DIRECTORY)
  # 确保头文件被正确安装到项目的include目录中，并且保持了相对目录结构
  install(
    FILES
      ${HDR}
    DESTINATION
      include/${INSTALL_DIR}
  )
endforeach()
