# Git
## 如何同步原有仓库的代码到自己 fork 出来的仓库
首先给自己的仓库添加一个 `upstream` ：

```shell
git remote add upstream https://github.com/whoever/whatever.git
```

拉取 `upstream` 的所有分支：

```shell
git fetch upstream
```

切换到 `master` 分支：

```shell
git checkout master
```

然后进行 `rebase` ，不使用 `merger` ，避免污染提交记录 ：

```shell
git rebase upstream/master
```

进行 `push` ：

```shell
git push -f origin master
```
