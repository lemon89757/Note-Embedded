# 主机端 ubuntu 环境配置
## 文件夹和磁盘用量查看
> 1. [Ubuntu下查看文件、文件夹和磁盘空间的大小](https://blog.csdn.net/BigData_Mining/article/details/88998472)

### 查看文件夹
1. 查看当前目录已经使用总大小以及当前目录下一级文件夹各自使用的总空间大小
    ```bash
    du -h --max-depth=1
    ```
2. 查看当前文件夹总量
    ```bash
    du -sh
    ```
3. 查看当前文件夹下所有文件（递归子目录中的文件）
    ```bash
    dh -h
    ```
4. 查看指定文件夹的总量
    ```bash
    du -h dir-name # 同样会显示该文件夹下所有文件的大小
    du -hs dir-name # 统计 summarize
    ```
5. 其他：`du --help`

### 查看磁盘
```bash
df -h
```

## 系统中存在多个软件版本时的默认选择
通过 `update-alternatives` 工具对 Linux 系统中的软件版本进行管理、切换。

`update-alternatives` 只管理系统链接，不会卸载实际的软件包（如果需要卸载：`sudo apt-get remove package-name`）。

1. install
    ```bash
    # update-alternatives: --install needs <link> <name> <path> <priority>
    sudo update-alternatives --install /usr/bin/g++  g++ /usr/bin/g++-5 50
    ```
   - `update-alternatives`：Debian/Ubuntu 系统的替代版本管理工具；
   - `--install`：注册一个新的替代选项；
   - `/usr/bin/g++`：系统默认的 g++ 路径（对外暴露的统一入口）；
   - `g++`：替代组的名称；
   - `/usr/bin/g++-5`：实际指向的具体版本（`g++ 5.x`）；
   - `50`：优先级数值（越大优先级越高）。
   - 上述命令的作用是将 `g++-5` 注册为 `g++` 的一个替代版本。**当系统中有多个 `g++` 版本时，用户可以通过统一命令切换默认使用的编译器，而无需手动修改软链接**。

2. remove
    ```bash
    sudo update-alternatives --remove g++ /usr/bin/g++-5
    ```
   - 从 `g++` 替代组中移除 `g++-5` 这个选项；
   - 如果这是唯一/最后一个选项，直接删除真个替代组。
    ```bash
    sudo update-alternatives --remove-all g++
    ```
   - 完全删除 `g++` 的替代配置。

3. display
    ```bash
    update-alternatives --display g++
    ```
   - 查看当前配置

4. 自动/手动模式
    ```bash
    # 改为自动模式（按优先级自动选择）
    sudo update-alternatives --auto g++

    # 手动指定默认版本
    sudo update-alternatives --config g++
    ```

## 交叉编译器
1. 交叉编译器安装
   - 下载
    [Linaro 交叉编译器下载](https://releases.linaro.org/components/toolchain/binaries/4.9-2017.01/arm-linux-gnueabihf/)
   - 安装
    ```bash
    sudo mkdir /usr/local/arm
    # 解压
    sudo tar -xf gcc-linaro-xxx...tar.xz -C /usr/local/arm/
    # /etc/profile （建议使用时单独导出，创建 .sh 脚本即可）
    sudo vi /etc/profile
    export PATH=$PATH:/usr/local/arm/gcc-linaro-.../bin # 添加到文件末尾，并重启系统
    # 可以添加到 .bashrc 文件末尾，并 source ~/.bashrc 即可
    ```
   - 依赖安装
    ```bash
    sudo apt-get install lsb-core lib32stdc++6
    ```
   - 验证
    ```bash
    # 在导入环境变量的终端中：
    arm-linux-gnueabihf-gcc -v
    ```
2. clangd
   - 生成 `compile_commands.json`：
        ```bash
        # Makefile
        sudo apt-get install bear
        make clean; bear -- make               # for bear 3.0.x versions
        make clean; bear -- make               # for bear 2.4.x versions

        # CMake
        cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=1
        ```
   - *clangd* 宏展开的 *expand to* 功能要求：*clangd* 版本大于 16

3. 工具链头文件、库文件目录查看
    ```bash
    # 提前设置环境变量，使系统能够找到编译工具
    echo 'main(){}' | arm-linux-gnueabihf-gcc -E -v -
    ```

## 网络登录
开启 ubuntu 的 ssh 服务后就可以通过 ssh 登录到 ubuntu 主机端：
```bash
sudo apt-get install openssh-server
```
ssh 的配置文件为 `/etc/ssh/sshd_config`，使用默认配置即可。


## 网络下载
### nfs
1. 安装 nfs 服务：
    ```bash
    sudo apt-get install nfs-kernel-server rpcbind
    ```
2. 创建共享文件夹供 nfs 服务器使用：`/path/to/nfs-dir`
3. 设置共享文件夹
   ```bash
   sudo vim /etc/exports
   # 在 /etc/exports 中添加如下内容
   /path/to/nfs-dir *(rw,sync,no_root_squash)
   ```
4. 重启 nfs 服务
   ```bash
   sudo /etc/init.d/nfs-kernel-server restart
   ```

### tftp
1. 安装 tftp-hpa 和 tftpd-hpa
    ```bash
    sudo apt-get install tftp-hpa tftpd-hpa
    sudo apt-get install xinetd
    ```
2. 创建共享文件夹供 tftp 使用：`/path/to/tftp-dir` 并**提供权限**：
   ```bash
   mkdir /path/to/tftp-dir
   chmod 777 /path/to/tftp-dir
   ```
3. 配置 tftp
   - 新建文件 `/etc/xinetd.d/tftp`，在里面输入如下内容：
        ```txt
        server tftp
        {
            socket_type = dgram
            protocol = udp
            wait = yes
            user = root
            server = /usr/sbin/in.tftpd
            server_args = -s /path/to/tftp-dir
            disable = no
            per_source = 11
            cps = 100 2
            flags = IPv4
        }
        ```
    - 重启 tftp 服务
        ```bash
        sudo service tftpd-hpa start
        ```
    - 修改 `/etc/default/tftpd-hpa` 文件：
        ```txt
        # /etc/default/tftpd-hpa

        TFTP_USERNAME="tftp"
        TFTP_DIRECTORY="/path/to/tftp-dir"
        TFTP_ADDRESS=":69"
        TFTP_OPTIONS="-l -c -s"
        ```
4. 重启 tftp 服务器
    ```bash
    sudo service tftpd-hpa restart
    ```
注意：后续需要通过 tftp 传输的文件，放置在 /path/to/nfs-dir 后还需要给予文件相应的权限。