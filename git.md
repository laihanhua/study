## 合并commit
[戳👇](https://segmentfault.com/a/1190000007748862)

## cherry-pick(包括单个commit和分支mr合并处理)
[戳👇](https://www.ruanyifeng.com/blog/2020/04/git-cherry-pick.html)

## 摘除commit
1. 找到要摘除commit的前一条commit_id
2. git rebase -i commit_id
3. 在编辑页面中，把摘除的commit_id设置为drop，保留的commit_id设置为pick
4. 保存退出
