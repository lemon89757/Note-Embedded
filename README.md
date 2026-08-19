# 记录嵌入式 Linux 驱动开发学习过程
## 文档格式
1. 使用行内代码符号“`行内代码`”包含的内容：
   - 变量名，`a`
   - 函数名，`func()`
   - 表达式, `a += 1;`
   - 类型名, `struct spi_message`
   - 路径, `Documentation/...`
2. 函数名后统一带有“()”后缀，如 `spi_async()`。
3. 说明字符，如 spi 子系统中的 spi_master、spi_message、spi_transfer 中可能将其简写为 master、message (msg)、transfer（指示某类对象而不是某一个体实例），使用斜体（“*斜体*”）表示。
4. 重点内容使用“**重点内容**”表示。
5. 多行代码块需显式说明语言格式，如：
    ```c
    // this is a block of c language
    ```
6. 文档内标题等级一般不应该超过 4 级，且只有一个 1 级标题。
