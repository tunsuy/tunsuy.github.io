git 常用操作

1、远程分支代码覆盖本地分支代码
git checkout filename/dirname

2、远程其他分支覆盖本地分支代码
git checkout remote_branch_name -- filename/dirname

3、合并任意commit提交
git cherry-pick commit_id

4、合并其他分支
git merge branch_name

5、取消合并
git merge --abort
注：其他操作类似

6、回滚
git reset commit_id

7、拉去master分支
git checkout origin/master
git checkout -b master