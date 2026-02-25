# Note for Shell
1. `shell` 脚本要求：
   ```bash
   #!/bin/bash
   ...
   ```
2. 新增环境变量：
    ```bash
    export PATH=$PATH:/path/to/new/dir/
    ```
    需要注意的是，如果将上述内容保存到 `shell` 脚本中（`.sh`），如果想要在当前环境下导入上述环境变量，需要在执行：
    ```bash
    source xxx.sh
    ```
    详情可见《鸟哥 Linux 私房菜》（默认在子进程下执行脚本文件）