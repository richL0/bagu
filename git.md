https://www.atlassian.com/git

![[Pasted image 20240516235715.png]]

### cherry-pick
选一个commit ID，commit到当前branch

```shell
git cherry-pick 41a3da4
git cherry-pick 41a3da4 3a3bd31
```

### checkout
切换分支，切换之前一定要先commit，否则报错
```shell
git checkout newBranch
```

-f 强制commit，可能会丢失修改内容

### rebase
![[01 What is git rebase.svg]]

把当前branch的base重新设置
```shell
git rebase master
```

### revert
回滚某个commit，并再commit一次
revert和reset的区别：
![[04 (2).svg]]


### tag
对某个commit做一个标记
```shell
git tag v1.0.0 # add a tag
git tag -d v1.0.0 # delete a tag
git tag v1.0.1 -a -m 'message' 23d4ab4 # add a tag to 23d4ab4 with message 
```