# 开发板环境配置
## 网络配置
1. uboot 网络参数设置
   建议开发板和主机都连接到同一个路由器上。需要设置的参数：
   - `ipaddr`：开发板 ip 地址，可以不设置，使用 dhcp 命令来从路由器获取 ip 地址；
   - `ethaddr`：开发板的 MAC 地址，一定要设置；
   - `gatewayip`：网关地址；
   - `netmask`：子网掩码；
   - `serverip`：服务器 ip 地址，也就是 ubuntu 主机 ip 地址。

    示例：
    ```bash
    setenv ipaddr 192.168.1.50
    setenv ethaddr b8:ae:1d:01:00:00
    setenv gatewayip 192.168.1.1
    setenv netmask 255.255.255.0
    setenv serverip 192.168.1.253
    saveenv
    ```
    - 确保 ubuntu 主机和开发板的 ip 地址在同一个网段内；
    - 确保同一个网段内设备的 ethaddr 是不同的。

2. 挂载网络文件系统：
   ```bash
   mount -t nfs -o nolock,vers=3 server-ip:/path/to/nfs-dir /path/to/mount
   ```

## 软件移植
### tslib
- 源码下载
   [github 地址](https://github.com/kergoth/tslib)，将其放置 ubuntu 主机上。
- 依赖安装
   ```bash
   sudo apt-get update
   sudo apt install automake libtool m4 autoconf pkg-config
   ```
- 源码编译
   ```bash
   tar xf tslib-...tar.bz2
   cd tslib-...
   ./autogen.sh      # 生成 Makefile
   # 先导出交叉编译器相关环境变量
   # 进行 tslib 相关配置
   ./configure --host=arm-linux-gnueabihf ac_cv_func_malloc_0_nonnull=yes --cache-file=arm-linux.cache -prefix=/path/to/out
   # 编译
   make
   # 安装到 /path/to/out
   make install
   ```
- 移植到文件系统
   ```bash
   # 打包
   tar -jcf ./package-name.tar.bz2 /path/to/out
   # 拷贝到开发板根文件系统
   # 解压至 /usr/lib
   tar xf package-name.tar.bz2 -C /usr/lib
   ```
- 配置 `tslib` 环境变量
   ```bash
   # /etc/profile 文件
   export TSLIB_ROOT=/usr/lib/package-name
   export TSLIB_CONSOLEDEVICE=none
   export TSLIB_FBDEVICE=/dev/fb0
   export TSLIB_TSDEVICE=/dev/input/event1
   export TSLIB_CONFFILE=$TSLIB_ROOT/etc/ts.conf
   export TSLIB_PLUGINDIR=$TSLIB_ROOT/lib/ts
   # 电阻屏专用
   # 电容屏可不用加这项，如果加了，使用 ts_calibrate 校准后会生成 /etc/pointercal 文件，请删除它，否则触摸不准确，因为电容屏不需要校准
   export TSLIB_CALIBFILE=/etc/pointercal
   export LD_PRELOAD=$TSLIB_ROOT/lib/libts.so
   # 使能 source /etc/profile，下次开机自动使能
   ```
- 测试
   ```bash
   /usr/lib/package-name/bin/ts_test
   ```

### Qt-5.12.9
> 1. [正点原子i.mux6ull开发板ubuntu22.04环境下QT5.12.9移植问题](https://blog.csdn.net/qq_44306158/article/details/140175855?csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22140175855%22%2C%22source%22%3A%22qq_44306158%22%7D%20%E5%8F%91%E5%87%BA%E5%89%8D%E8%BF%98%E5%9C%A8%E5%AE%A1%E6%A0%B8)

- 源码下载
   [Qt-5.12.9](https://download.qt.io/archive/qt/5.12/5.12.9/single/)
  - 在 Windows 上下载好拷贝到 Ubuntu 主机上。
  - 在 Ubuntu 主机上通过 `wget` 下载：
      ```bash
      wget https://download.qt.io/archive/qt/5.12/5.12.9/single/qt-everywhere-src-5.12.9.tar.xz
      ```
- 解压
   ```bash
   tar xf qt-everywhere-src-5.12.9.tar.xz
   ```
- 修改 `qmake.conf`
   ```Makefile
   # 位于 qt 源码路径的 /mkspecs/qws/linux-arm-gnueabi-g++/qmake.conf

   #
   # qmake configuration for building with arm-linux-gnueabi-g++
   #
   MAKEFILE_GENERATOR = UNIX
   CONFIG += incremental
   QMAKE_INCREMENTAL_STYLE = sublib

   # 需要修改的地方
   QT_QPA_DEFAULT_PLATFORM = linuxfb
   QMAKE_CFLAGS += -O2 -march=armv7-a -mtune=cortex-a7 -mfpu=neon -mfloat-abi=hard
   QMAKE_CXXFLAGS += -O2 -march=armv7-a -mtune=cortex-a7 -mfpu=neon -mfloat-abi=hard


   include(../common/linux.conf)
   include(../common/gcc-base-unix.conf)
   include(../common/g++-unix.conf)
   
   # 需要修改的地方
   # modifications to g++.conf
   QMAKE_CC = arm-linux-gnueabihf-gcc
   QMAKE_CXX = arm-linux-gnueabihf-g++
   QMAKE_LINK = arm-linux-gnueabihf-g++
   QMAKE_LINK_SHLIB = arm-linux-gnueabihf-g++

   # 需要修改的地方
   # modifications to linux.conf
   QMAKE_AR = arm-linux-gnueabihf-ar cqs
   QMAKE_OBJCOPY = arm-linux-gnueabihf-objcopy
   QMAKE_NM = arm-linux-gnueabihf-nm -P
   QMAKE_STRIP = arm-linux-gnueabihf-strip
   load(qt_config)
   ```
- 配置编译选项：
   ```bash
   ./configure -help # 查看可配置选项
   ```
   - 编辑脚本文件如下：
   ```shell
   ./configure -prefix /path/to/install \
   -opensource \
   -confirm-license \
   -release \
   -strip \
   -shared \
   -xplatform linux-arm-gnueabi-g++ \
   -optimized-qmake \
   -c++std c++11 \
   --rpath=no \
   -pch \
   -skip qt3d \
   -skip qtactiveqt \
   -skip qtandroidextras \
   -skip qtcanvas3d \
   -skip qtconnectivity \
   -skip qtdatavis3d \
   -skip qtdoc \
   -skip qtgamepad \
   -skip qtlocation \
   -skip qtmacextras \
   -skip qtnetworkauth \
   -skip qtpurchasing \
   -skip qtremoteobjects \
   -skip qtscript \
   -skip qtscxml \
   -skip qtsensors \
   -skip qtspeech \
   -skip qtsvg \
   -skip qttools \
   -skip qttranslations \
   -skip qtwayland \
   -skip qtwebengine \
   -skip qtwebview \
   -skip qtwinextras \
   -skip qtx11extras \
   -skip qtxmlpatterns \
   -make libs \
   -make examples \
   -nomake tools -nomake tests \
   -gui \
   -widgets \
   -dbus-runtime \
   --glib=no \
   --iconv=no \
   --pcre=qt \
   --zlib=qt \
   -no-openssl \
   --freetype=qt \
   --harfbuzz=qt \
   -no-opengl \
   -linuxfb \
   --xcb=no \
   -tslib \
   --libpng=qt \
   --libjpeg=qt \
   --sqlite=qt \
   -plugin-sql-sqlite \
   -I/path/to/tslib/include \
   -L/path/to/tslib/lib \
   -recheck-all
   ```
  - 通过脚本进行配置
   ```bash
   chmod +x autoconfig.sh
   # 安装 g++
   sudo apt-get install g++
   # 配置
   ./autoconfig.sh
   # 输出 "Qt is now configured for building. Just run 'make'" 表示可以进行后续编译
   ```
- 编译并安装
   ```bash
   time (make -j24) # 测试编译时间
   # 编译完进行安装
   make install
   # 查看安装后的内容
   ls /path/to/install
   ```
- 移植到文件系统
   ```bash
   # 打包
   tar -jcf ./package-name.tar.bz2 /path/to/install
   # 拷贝至开发板根文件系统
   # 解压
   tar xf package-name.tar.bz2 -C /usr/lib
   ```
- 配置 Qt5 的环境变量
   ```bash
   # 编辑 /etc/profile，在末尾添加内容：
   export QT_ROOT=/usr/lib/package-name
   export QT_QPA_GENERIC_PLUGINS=tslib:/dev/input/event1
   export QT_QPA_FONTDIR=/usr/share/fonts
   export QT_QPA_PLATFORM_PLUGIN_PATH=$QT_ROOT/plugins
   export QT_QPA_PLATFORM=linuxfb:tty=/dev/fb0
   export QT_PLUGIN_PATH=$QT_ROOT/plugins
   export LD_LIBRARY_PATH=$QT_ROOT/lib:$QT_ROOT/plugins/platforms
   export QML2_IMPORT_PATH=$QT_ROOT/qml
   export QT_QPA_FB_TSLIB=1
   ```
   - 要想 Qt 程序显示中文，将 Windows 下的（`C:\Windows\Fonts`）下的中文字库放置 `/usr/share/fonts` 下（注意：字库有版权）。
- 测试
   ```bash
   source /etc/profile # 或者重启
   # 运行示例程序
   /usr/lib/package-name/examples/widgets/animation/animatedtiles/animatedtiles
   ```
   - Qt 官方说明需要 `libudev` 才能支持热插拔。