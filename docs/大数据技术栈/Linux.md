## 文件系统常见命令

以下是一些常见的 Linux 文件系统相关操作的命令：

1. `ls`：列出目录中的文件和子目录。

2. `cd`：切换当前工作目录。

3. `pwd`：显示当前工作目录的路径。

4. `mkdir`：创建新目录。

5. `rmdir`：删除空目录。

6. `cp`：复制文件或目录。

7. `mv`：移动文件或目录，也可用于重命名文件或目录。

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

Linux 中的权限由三个基本权限组成：读取权限（Read，简写为  `r`）、写入权限（Write，简写为  `w`）和执行权限（Execute，简写为  `x`）。

这些权限可以应用于**三个不同的用户类别**：文件所有者（Owner）、文件所属组（Group）和其他用户（Others）。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309071718538.png)

在 Linux 中第一个字符代表这个文件是目录、文件或链接文件等等。

- 当为  `d`  则是目录
- 当为  `-`  则是文件；
- 若是  `l`  则表示为链接文档 (link file)；
- 若是  `b`  则表示为装置文件里面的可供储存的接口设备 (可随机存取装置)；
- 若是  `c`  则表示为装置文件里面的串行端口设备，例如键盘、鼠标 (一次性读取装置)。

Linux 文件属性有两种设置方法，一种是数字，一种是符号。

Linux 文件的基本权限就有九个，分别是  **owner/group/others(拥有者/组/其他)**  三种身份各有自己的  **read/write/execute**  权限。

若文件的权限字符为：`-rwxrwxrwx` ，其中，我们可以使用数字来代表各个权限，各权限的分数对照表如下：

- r: 4
- w: 2
- x: 1

每种身份 (owner/group/others) 各自的三个权限 (r/w/x) 分数是需要累加的，例如当权限为： `-rwxrwx---`  分数则是：

- owner = rwx = 4+2+1 = 7
- group = rwx = 4+2+1 = 7
- others = --- = 0+0+0 = 0

## Shell 编程

[Bash scripting cheatsheet (devhints.io)](https://devhints.io/bash)

Example:

```bash
#!/bin/bash
echo "Today is " `date`

echo -e "\nenter the path to directory"
read the_path

echo -e "\n you path has the following files and folders: "
ls $the_path
```

```bash
chmod u+x run_all.sh
sh run_all.sh
```

### 条件控制

```bash
if [[ condition ]];
then
 statement
elif [[ condition ]]; then
 statement
else
 do this by default
fi
```

```bash
#!/bin/bash
echo "Please enter a number: "
read num

if [ $num -gt 0 ]; then
  echo "$num is positive"
elif [ $num -lt 0 ]; then
  echo "$num is negative"
else
  echo "$num is zero"
fi
```

### 循环

```bash
#!/bin/bash
i=1
while [[ $i -le 10 ]] ; do
   echo "$i"
  (( i += 1 ))
done
```

```bash
for ((i = 0 ; i < 100 ; i++)); do
  echo "$i"
done
```

```bash
for i in {1..5}; do
    echo "Welcome $i"
done
```

## grep

`grep` 是一个在 Linux 系统中用于搜索文本的强大命令。它可以根据指定的模式（正则表达式）在文件中查找匹配的行，并将它们输出到终端。

以下是 `grep` 命令的一般语法：

```bash
grep options pattern file
```

- `options` 是可选的参数，用于指定 `grep` 命令的不同选项和标志。

- `pattern` 是要搜索的模式，可以是普通字符串或正则表达式。

- `file` 是要在其中进行搜索的文件名或文件列表。如果省略文件名，则 `grep` 会从标准输入读取数据。

下面是一些常见的 `grep` 选项和用法示例：

- 在单个文件中搜索匹配的行：

```bash
grep "pattern" file.txt
```

- 在多个文件中搜索匹配的行：

```bash
grep "pattern" file1.txt file2.txt
```

- 在目录及其子目录中递归搜索匹配的行：

```bash
grep -r "pattern" directory/
```

- 忽略匹配模式的大小写：

```bash
grep -i "pattern" file.txt
```

- 显示匹配行之前的几行：

```bash
grep -B 2 "pattern" file.txt
```

- 显示匹配行之后的几行：

```bash
grep -A 2 "pattern" file.txt
```

- 显示匹配行及其上下文的几行：

```bash
grep -C 2 "pattern" file.txt
```

- 使用正则表达式进行高级模式匹配：

```bash
grep "[0-9]+[A-Z]+" file.txt
```

- 反向搜索，显示不匹配模式的行：

```bash
grep -v "pattern" file.txt
```

- 统计匹配的行数：

```bash
grep -c "pattern" file.txt
```

## sed

`sed`（Stream Editor）是一个流式文本编辑器，用于对文本进行转换、替换和修改。它可以在文件处理管道中使用，也可以对文件进行直接编辑。下面是一些常见的 `sed` 命令和用法示例：

- 替换文本中的字符串：

  ```bash
  sed 's/old_string/new_string/' file.txt
  ```

- 替换文本中的所有匹配项：

  ```bash
  sed 's/old_string/new_string/g' file.txt
  ```

- 替换文本中的第 N 个匹配项：

  ```bash
  sed 's/old_string/new_string/N' file.txt
  ```

- 删除文本中的匹配行：

  ```bash
  sed '/pattern/d' file.txt
  ```

- 删除文本中的空行：

  ```bash
  sed '/^$/d' file.txt
  ```

- 在指定行之前或之后插入文本：

  ```bash
  sed '3i\inserted_text' file.txt   # 在第 3 行之前插入文本
  sed '5a\appended_text' file.txt   # 在第 5 行之后插入文本
  ```

- 按模式进行文本截取和提取：

  ```bash
  sed -n '2,5p' file.txt   # 提取第 2 到 5 行的文本
  sed -n '/pattern/p' file.txt   # 提取匹配模式的行
  ```

- 根据正则表达式进行高级模式匹配和替换：

  ```bash
  sed 's/[0-9]\+/replacement/' file.txt   # 替换数字为指定字符串
  sed '/pattern/ s/replace_string/replacement/g' file.txt   # 仅在匹配行中替换
  ```

- 从文件中读取 `sed` 命令：

  ```bash
  sed -f script.sed file.txt
  ```

这些是 `sed` 命令的一些常见用法示例，涵盖了文本替换、删除、插入、截取和高级模式匹配等功能。`sed` 命令非常强大且灵活，可以通过组合和使用不同的选项和命令实现各种文本转换和编辑操作。

## awk

Awk 是一种强大的文本处理工具，广泛用于 Linux 系统中的命令行环境。它提供了一种简洁而灵活的方式来对文本文件进行分析、处理和转换。

Awk 的名字是由其三位创始人（Alfred Aho、Peter Weinberger 和 Brian Kernighan）的姓氏首字母组成的。它的工作方式是逐行读取输入文本，将每行分割成字段，并根据用户定义的规则执行相应的操作。

下面是一些 Awk 的常用特性和用法：

- 模式与动作：Awk 的基本结构是模式与动作的组合。模式用于选择感兴趣的行，而动作则是在选定的行上执行的操作。如果未提供模式，则动作将适用于所有行。

```bash
awk '/pattern/ { action }' file.txt
```

- 字段分割：Awk 默认将每行按空格或制表符分割成字段，并将字段存储在变量 `$1`, `$2`, `$3` 等中。可以使用 `-F` 选项指定不同的字段分隔符。

```bash
awk -F ',' '{ print $1 }' file.csv
```

- 内置变量：Awk 提供了许多内置变量，用于访问输入文本的各种属性，如行号 (`NR`)、字段数 (`NF`) 等。

```bash
awk '{ print NR, NF }' file.txt
```

- 条件判断：Awk 支持条件语句，可以根据条件执行不同的动作。

```bash
awk '{ if ($1 > 10) print $0 }' file.txt
```

- 控制结构：Awk 具有类似于循环和分支语句的控制结构，可以用于迭代处理文本和进行条件判断。

```bash
awk '{ for (i = 1; i <= NF; i++) print $i }' file.txt
```

- 内置函数：Awk 提供了许多内置函数，可以用于执行各种操作，如字符串处理、数学计算等。

```bash
awk '{ print length($0) }' file.txt
```

Awk 是一种非常强大而灵活的工具，可以根据不同的需求进行文本处理和数据分析。它在命令行环境中广泛应用于数据提取、报告生成、日志分析等各种任务。在 Linux 中，Awk 通常与其他命令（如 grep、sed 和 sort）以及重定向和管道操作符结合使用，以实现更复杂的文本处理任务。
