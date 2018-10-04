# 常用Git命令操作 #

记录一些常用/实用的Git命令

1. 
```
git clone <https/ssh/git>
```
克隆远程项目。

2. 
```
git diff
```
此命令比较的是工作目录中当前文件和暂存区域快照之间的差异，也就是修改之后还没有暂存起来的变化内容。请注意，单单 git diff 不过是显示还没有暂存起来的改动，而不是这次工作和上次提交之间的差异。所以有时候你一下子暂存了所有更新过的文件后，运行 git diff 后却什么也没有，就是这个原因。

3. 
```
git diff --cached
git diff --staged
```
比较已经暂存起来的文件和上次提交时的快照之间的差异。

4. 
```
git commit
git commit -m 'message'
```
提交。记住，提交时记录的是放在暂存区域的快照，任何还未暂存的仍然保持已修改状态，可以在下次提交时纳入版本管理。每一次运行提交操作，都是对你项目作一次快照，以后可以回到这个状态，或者进行比较。

5. 
```
git commit -a
```
自动把所有**已经跟踪过的文件**暂存起来一并提交，从而跳过 git add 步骤。

6. 
```
rm /path/to/file
git rm /path/to/file
```
要从 Git 中移除某个文件，就必须要从已跟踪文件清单中移除（确切地说，是从暂存区域移除），然后提交。可以用 git rm 命令完成此项工作，**并连带从工作目录中删除指定的文件，这样以后就不会出现在未跟踪文件清单中了**。如果删除之前修改过并且已经放到暂存区域的话，则必须要用强制删除选项 -f（译注：即 force 的首字母），以防误删除文件后丢失修改的内容。

7. 
```
git rm --cached /path/to/file
git rm log/\*.log // 删除所有 log/ 目录下扩展名为 .log 的文件
git rm \*~ // 递归删除当前目录及其子目录中所有 ~ 结尾的文件
```
想把文件从 Git 仓库中删除（亦即从暂存区域移除），但仍然希望保留在当前工作目录中。换句话说，仅是从跟踪清单中删除。

8. 
```
git mv file_from file_to
```
移动文件。运行 git mv 就相当于运行了下面三条命令：
```
mv file_from file_to
git rm file_from
git add file_to
```
用其他工具（批处理）改名的话，要**记得在提交前删除老的文件名，再添加新的文件名**。

9. 
```
git log                  // 默认不用任何参数的话，git log 会按提交时间列出所有的更新，最近的更新排在最上面。 
git log -p -2            // -p 选项展开显示每次提交的内容差异，-2 仅显示最近的两次更新。 
git log -p --word-diff   // 单词层面的对比，比行层面的对比，更加容易观察
git log --state          // 仅显示简要的增改行数统计
git log --pretty=oneline // 指定使用完全不同于默认格式的方式展示提交历史。比如用 oneline 将每个提交放在一行显示，这在提交数很大时非常有用。另外还有 short，full 和 fuller 可以用。
git log --pretty=format:"%h - %an, %ar : %s" // format可以定制要显示的记录格式，这样的输出便于后期编程提取分析
git log --pretty="%h - %s" --author=gitster --since="2008-10-01" --before="2008-11-01" --no-merges -- t/ // 查看 Git 仓库中，2008 年 10 月期间，Junio Hamano 提交的但未合并的测试脚本（位于项目的 t/ 目录下的文件）
```
[查看提交历史](https://git-scm.com/book/zh/v1/Git-%E5%9F%BA%E7%A1%80-%E6%9F%A5%E7%9C%8B%E6%8F%90%E4%BA%A4%E5%8E%86%E5%8F%B2)。


参考 [git-scm](https://git-scm.com/book/zh/v1)

> 欢迎转载，注明出处即可。