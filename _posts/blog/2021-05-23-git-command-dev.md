
放弃`git add`放入暂存区的指定文件`filename`：

`git reset HEAD filename`

放弃暂存区所有文件：

`git reset HEAD .`

执行后所有更改文件不会丢失，相当于撤销了`git add`命令。

这时如果我们不想要本地修改的文件，就可以使用：

`git checkout -- filename`

也可以全部丢弃修改：

`git checkout .`

`git commit`之后还没有执行`git push`,这时想要撤销，可以使用：

`git reset --soft HEAD~1`

这时所有本次`commit`的文件都会回到暂存区。

如果只是在`git commit`之后想要修改提交信息的话：

`git commit --amend`