# clickhouse源码构建

{:toc}

* TOC

## 概述
- 官方文档：https://clickhouse.com/docs/en/development/build/
- 源码分支 21.12
- 环境依赖
  - ubuntu: 16.04
  - cmake: 3.20.6
  - clang: clang-13

## 步骤

- 准备源码: 

  clickhouse官方库为：https://github.com/ClickHouse/ClickHouse

  `git clone -b 21.12 https://github.com/ClickHouse/ClickHouse.git`，可切换到其他具体的分支

- 执行git submodule更新相关的依赖包，依赖包会下载到``ClickHouse/contrib`下

  ```shell
  git submodule sync
  # 初始化并下载相关依赖
  git submodule update --init --recursive 
  ```

- 编译环境下载相关的依赖，如cmake，clang等

- 将下面脚本`build.sh`放在ClickhouseHouse目录下，执行`bash build.sh`

  ```
  mkdir build-clang
  cd build-clang
  
  je=1
  tc=1
  BT=Release
  #BT=Debug
  
  export TMPDIR=/data/tmp
  /opt/cmake-3.20.6-linux-x86_64/bin/cmake .. -DENABLE_JEMALLOC=${je}  -DENABLE_TCMALLOC=${tc} -DCMAKE_EXPORT_COMPILE_COMMANDS=1  -DENABLE_TESTS=OFF  -DCMAKE_BUILD_TYPE=${BT} -DENABLE_RDKAFKA=0 -DCMAKE_CXX_COMPILER=`which clang++-13` -DCMAKE_C_COMPILER=`which clang-13`
  
  if [ $? -ne 0 ]; then
    echo "cmake faild"
    exit 1
  fi
  
  
  ninja -j20
  
  
  if [ $? -ne 0 ]; then
    echo "faild"
    exit 1
  fi
  ```

- 编译完可执行文件在`build-clang/programs下`

  ```
  $ tree -L 1 programs
  programs
  ├── bash-completion
  ├── benchmark
  ├── clickhouse
  ├── clickhouse-benchmark -> clickhouse
  ├── clickhouse-client -> clickhouse
  ├── clickhouse-compressor -> clickhouse
  ├── clickhouse-copier -> clickhouse
  ├── clickhouse-extract-from-config -> clickhouse
  ├── clickhouse-format -> clickhouse
  ├── clickhouse-git-import -> clickhouse
  ├── clickhouse-keeper -> clickhouse
  ├── clickhouse-keeper-converter -> clickhouse
  ├── clickhouse-library-bridge
  ├── clickhouse-local -> clickhouse
  ├── clickhouse-obfuscator -> clickhouse
  ├── clickhouse-odbc-bridge
  ├── clickhouse-server -> clickhouse
  ├── clickhouse-static-files-disk-uploader -> clickhouse
  ├── client
  ├── CMakeFiles
  ├── cmake_install.cmake
  ├── compressor
  ├── copier
  ├── CTestTestfile.cmake
  ├── extract-from-config
  ├── format
  ├── git-import
  ├── install
  ├── keeper
  ├── keeper-converter
  ├── library-bridge
  ├── local
  ├── obfuscator
  ├── odbc-bridge
  ├── server
  └── static-files-disk-uploader
  ```

## 碰到的问题

各种包依赖，这个时候看报错日志，把相关依赖解决就行