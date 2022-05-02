@[TOC](使用sed命令来高效处理文本)

sed命令的格式为

```bash
sed [options] 'command' file(s)  # 以command指令来处理file(s)
sed [options] -f script file(s)  # 以script脚本来处理file(s)
```

其中，options主要常用到下面几种：


**-i**：直接修改原文件，而不是输出到屏幕

```bash
sed -i 's/test/TEST' file01
```

**-n**：即`--quiet`，仅显示处理后的结果，而不是文件的全部内容
与p指令一起使用可以只打印发生替换的行。
```bash
sed -n 's/test/TEST/p' file01
```

**-e** ：即`--expression`，允许在同一行里执行多条命令
```bash
sed -e '1,5d' -e 's/test/check/' file01
```

# 定界符
sed一般使用`/`作为定界符，也可以使用其他符号。定界符出现在要匹配的字符串中时，需要用反斜杠进行**转义**。

```bash
# 将test全部替换为TEST
sed 's/test/TEST/g' file01
sed 's#test#TEST#g' file01
sed 's:test:TEST:g' file01
sed 's|test|TEST|g' file01
# 将/bin全部替换为/usr/bin
sed 's/\/bin/\/usr\/bin/g' file01
sed 's#/bin#/usr/bin#g' file01
```

# 打印：p
与`-n`一起使用，只打印匹配到的行

```bash
# 打印匹配到test的行
sed -n '/test/p' file01
# 打印第99行以后所有内容
sed -n '100,$p' file02
```

# 读取：r
读取file02中的内容，添加到file01中匹配到test字符串的每一行之后
```bash
sed '/test/r file02' file01
```

# 写入：w
将file01中匹配到test字符串的所有行，写入到file02中（会覆盖文件中原有的内容）
```bash
sed '/test/w file02' file01
```

# 删除：d

```bash
# 删除空白行
sed '/^$/d' file01
# 删除第2行
sed '2d' file01
# 删除第2行到文件末尾的所有行
sed '2,$d' file01
# 删除最后一行
sed '$d' file01
# 删除test开头的所有行
sed '/^test/d' file01
```

# 替换：s
替换**每一行**中匹配到的**第一个**字符串：

```bash
sed 's/test/TEST/' file01
```

替换**每一行**中匹配到的**所有**字符串：
```bash
sed 's/test/TEST/g' file01
```

# 匹配
常用的匹配规则包括以下内容：

`^`：匹配行开始，如：`/^sed/`匹配所有以sed开头的行。
`$`：匹配行结束，如：`/sed$/`匹配所有以sed结尾的行。
`.`：匹配**一个**非换行符的任意字符，如：`/s.d/`匹配s后接一个任意字符，最后是d。
`*`：匹配**0个或多个**字符，如：`/*sed/`匹配所有模板是一个或多个空格后紧跟sed的行。
`\+`：匹配**1个或多个**字符，如：`/se\+d/`匹配s和d中间至少有一个字母e的字符串。
`[]`：匹配一个指定范围内的字符，如`/[sS]ed/`匹配sed和Sed。  
`[^]`：匹配一个不在指定范围内的字符，如：`/[^A-RT-Z]ed/`匹配不包含A-R和T-Z的一个字母开头，紧跟ed的行。
`\(..\)`：匹配子串，保存匹配的字符，如`s/\(love\)able/\1rs`，loveable被替换成lovers。
`\1`：子串匹配标记，与`\(..\)`配合使用。`\1`表示匹配到的第一个子串，以此类推，`\2`表示匹配到的第二个子串，...。
`&`：保存搜索字符用来替换其他字符，如`s/love/ **&** /`，love这成` **love** `。
`\<`：匹配单词的开始，如`:/\<love/`匹配包含以love开头的单词的行。
`\>`：匹配单词的结束，如`/love\>/`匹配包含以love结尾的单词的行。
`x\{m\}`：重复字符x，m次，如：`/0\{5\}/`匹配包含5个0的行。
`x\{m,\}`：重复字符x，至少m次，如：`/0\{5,\}/`匹配至少有5个0的行。
`x\{m,n\}`：重复字符x，至少m次，不多于n次，如：`/0\{5,10\}/`匹配5~10个0的行。
`[:space:]`：匹配单个空白符，包括空格、制表符等。


**示例：**
一个简单的示例

```bash
# \w\+匹配每一个单词，"&"给匹配到的字符串加上双引号
echo this is a test line | sed 's/\w\+/"&"/g'
"this" "is" "a" "test" "line"

# 匹配两个英文字母串，然后调换位置
echo aaa BBB | sed 's/\([a-z]\+\) \([A-Z]\+\)/\2 \1/'
BBB aaa
# \([a-z]\+\)匹配到aaa，标记为\1
# \([A-Z]\+\)匹配到BBB，标记为\2
```

一个复杂的示例
```bash
cat file03
#    logfile           /var/log/nscd.log
#    threads           4
     enable-cache      group               yes
     enable-cache      hosts               yes

# 去掉第一行的注释（定界符为/）
sed -i 's/^#\([[:space:]]*logfile[[:space:]]*\/var\/log\/nscd.log\)/\1/g' file03  
# 去掉第一行的注释（定界符为|）
sed -i 's|^#\([[:space:]]*logfile[[:space:]]*/var/log/nscd.log\)|\1|g' file03 
# 其中^#表示以#开头的行
# [[:space:]]*表示匹配0个或多个空白符
# \(...\)表示保留匹配子串，\1表示匹配到的第一个字串

cat file03
     logfile           /var/log/nscd.log
#    threads           4
     enable-cache      group               yes
     enable-cache      hosts               yes
     
# 将第三行的yes替换为no
sed -i 's/\([[:space:]]*enable-cache[[:space:]]*group[[:space:]]*\)yes/\1no/g' file03

cat file03
     logfile           /var/log/nscd.log
#    threads           4
     enable-cache      group               no
     enable-cache      hosts               yes
```



**References:**
【1】https://g-ghy.github.io/linux-command/c/sed.html
【2】https://www.twle.cn/c/yufei/sed/sed-basic-regular-expressions.html

