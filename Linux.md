## 文件系统常见命令

以下是一些常见的 Linux 文件系统相关操作的命令：

1. `ls`：列出目录中的文件和子目录。

2. `cd`：切换当前工作目录。

3. `pwd`：显示当前工作目录的路径。

4. `mkdir`：创建新目录。

5. `rmdir`：删除空目录。

6. `cp`：复制文件或目录。

   ```bash
   cp source_file destination_file
   ```

7. `mv`：移动文件或目录，也可用于重命名文件或目录。

   ```bash
   mv source_file destination_file
   ```

8. `rm`：删除文件或目录。

9. `touch`：创建空文件或更新文件的访问和修改时间戳。

10. `cat/head/tail/more/less`：将文件内容输出到终端。

11. `chmod`：修改文件或目录的权限。

    ```bash
    chmod permissions file_name
    chmod u+x file_name #给文件所有者添加执行权限
    ```

12. `chown`：修改文件或目录的所有者。

    ```bash
    chown user_name file_name
    ```

13. `chgrp`：修改文件或目录的所属组。

    ```bash
    chgrp group_name file_name
    ```

14. `find`：根据条件查找文件。

    ```bash
    find directory -name "filename_pattern"
    ```

15. `grep`：在文件中搜索匹配指定模式的文本。

    ```bash
    grep "pattern" file_name
    ```

16. `>` ：输出重定向（覆盖）

    ```bash
    echo "aaa" > a.txt
	```

 17. `>>` ：追加内容


## 打包和解压缩常见命令

```bash
tar -zcvf test.tar.gz 1.txt 2.txt 3.txt # 打包 + 压缩
tar -zxvf test.tar -C /home/destination # -C 指定目录解压
```

## Linux 文件权限

Linux 中的权限由三个基本权限组成：读取权限（Read，简写为 `r`）、写入权限（Write，简写为 `w`）和执行权限（Execute，简写为 `x`）。

这些权限可以应用于**三个不同的用户类别**：文件所有者（Owner）、文件所属组（Group）和其他用户（Others）。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309071718538.png)

在 Linux 中第一个字符代表这个文件是目录、文件或链接文件等等。

- 当为 `d` 则是目录
- 当为 `-` 则是文件；
- 若是 `l` 则表示为链接文档 (link file)；
- 若是 `b` 则表示为装置文件里面的可供储存的接口设备 (可随机存取装置)；
- 若是 `c` 则表示为装置文件里面的串行端口设备，例如键盘、鼠标 (一次性读取装置)。

Linux 文件属性有两种设置方法，一种是数字，一种是符号。

Linux 文件的基本权限就有九个，分别是 **owner/group/others(拥有者/组/其他)** 三种身份各有自己的 **read/write/execute** 权限。

若文件的权限字符为：`-rwxrwxrwx` ，其中，我们可以使用数字来代表各个权限，各权限的分数对照表如下：

- r: 4
- w: 2
- x: 1

每种身份 (owner/group/others) 各自的三个权限 (r/w/x) 分数是需要累加的，例如当权限为： `-rwxrwx---` 分数则是：

- owner = rwx = 4+2+1 = 7
- group = rwx = 4+2+1 = 7
- others = --- = 0+0+0 = 0

