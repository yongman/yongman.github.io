---
title: 清理git历史中的敏感信息
date: 2018-07-05 14:44:15
categories: [技术记录]
---

由于使用的编辑器带有代码头自动填充，或者不小心把密码明文写进了代码，并且不小心push到了github，当你发现的时候一般的补救措施是起不到作用的，除非...

1. 删除你的整个repo，这个对于维护了一段时间的项目肯定是无法接受的，删除后会导致repo中所有的数据丢失，包括issue和wiki，当然fork、star和watch也全部丢失了。
2. 修改密码，这个似乎不是在解决问题，而是在逃避问题。当然如果密码无法修改，那本方法也无效。
3. 重写历史。
对于这种方式官方提供两种方式。两种方式对于历史提交的SHAs都会更改，SHAs的修改对于活跃的社区项目可能会影响到其他开发者的pull request。所以建议在进行重写历史之前merge或者close掉待处理的pr。

- BFG

    bfg是一个jar包，其使用方式如下：

    直接将带有泄露信息的文件以及历史全部删除
    ```
    $java -jar bfg.jar --delete-files file_path_you_want_to_clean
    ```

    将泄露的信息进行文本替换成`***REMOVED***`
    ```
    $java -jar bfg.jar --replace-text sensitive_text_you_want_to_clean
    ```

    历史提交更新后，相应的信息在本地还没有彻底删除，执行下面的命令进行垃圾回收
    ```
    git reflog expire --expire=now --all && git gc --prune=now --aggressive
    ```

    最后执行`git push --force`

- git filter-branch

    在使用git filter-branch之前先把事前stash保存的工作pop出来，否则会导致无法恢复。这种方式似乎只能删除文件，不能直接进行文本替换
    ```
    git filter-branch --force --index-filter \
    'git rm --cached --ignore-unmatch PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA' \
    --prune-empty --tag-name-filter cat -- --all

    git push origin --force --all

    git for-each-ref --format='delete %(refname)' refs/original | git update-ref --stdin

    git reflog expire --expire=now --all
    git gc --prune=now
    ```


#### 参考
- [Removing sensitive data from a repository - User Documentation](https://help.github.com/articles/removing-sensitive-data-from-a-repository/)
- [BFG Repo-Cleaner by rtyley](https://rtyley.github.io/bfg-repo-cleaner/)


