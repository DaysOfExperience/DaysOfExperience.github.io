# 

# 1_常见指令以及权限理解

## Linux 基础指令

1. ls
2. pwd
3. cd    改变工作目录
4. touch touch命令参数可更改文档或目录的日期时间，包括存取时间和更改时间，或者新建一个不存在的文件。
5. mkdir [-p]
6. rm
7. man 1 是普通的命令 2 是系统调用,如open,write之类的 3 是库函数,如printf,fread
8. cp     cp [选项] 源文件或目录 目标文件或目录
9. mv  移动文件/目录   重命名文件/目录
10. cat  查看目标文件的内容
11. less  功能更强大的浏览文件内容的工具
12. head tail 
13. find find pathname -options -name  按照文件名查找文件
14. grep [选项] 搜寻字符串 文件 
     -i ：忽略大小写的不同，所以大小写视为相同 
    -n ：顺便输出行号 
    -v ：反向选择，亦即显示出没有 '搜寻字符串' 内容的那一行
15. zip [选项] 压缩文件.zip 目录或文件   -r 递归处理，将指定目录下的所有文件和子目录一并处理
16. tar  打包/解包   = =
    暂时略了

## Linux 扩展指令

1. top
2. ps
3. su  su [用户名]   切换用户  su root 时root可省略（Linux下的两种用户：超级用户root和普通用户）
4. who
5. chmod chown chgrp
6. netstat
7. ipconfig

## Shell命令及其运行原理（Bash）

## Linux权限
