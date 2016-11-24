---
date: 2016-11-23 23:03
title: atomic类
tags: [git,git-flow]
categories: [tool,git]
---

#Git flow 开发流程

-   branching model

model分成兩個主要分支，三種支援性分支

        主要分支
            master: 永遠處在 production-ready 狀態
            develop: 最新的下次發佈開發狀態

        支援性分支
            Feature branches: 開發新功能都從 develop 分支出來，完成後 merge 回 develop
            Release branches: 準備要 release 的版本，只修 bugs。從 develop 分支出來，完成後 merge 回 master 和 develop
            Hotfix branches: 等不及 release 版本就必須馬上修 master 趕上線的情況。會從 master 分支出來，完成後 merge 回 master 和 develop
---
-   create branching model

    git flow init
        Initialized empty Git repository in /Users/fannheyward/flowTest/.git/
        No branches exist yet. Base branches must be created now.
        Branch name for production releases: [master]
        Branch name for "next release" development: [develop]

        How to name your supporting branch prefixes?
        Feature branches? [feature/]
        Release branches? [release/]
        Hotfix branches? [hotfix/]
        Support branches? [support/]
        Version tag prefix? []
---

-   新功能开发, 代号 f1

    1.git flow feature start f1
        Switched to a new branch 'feature/f1'

        Summary of actions:
        - A new branch 'feature/f1' was created, based on 'develop'
        - You are now on branch 'feature/f1'

        Now, start committing on your feature. When done, use:

            git flow feature finish f1

        git-flow 从 develop 分支创建了一个新的分支 feature/f1，并自动切换到这个分支下面。然后就可以进行 f1 功能开发，中间可以多次的 commit 操作。

    2.git flow feature finish f1
        Switched to branch 'develop'
        Already up-to-date.
        Deleted branch feature/f1 (was 7bb5749).

        Summary of actions:
        - The feature branch 'feature/f1' was merged into 'develop'
        - Feature branch 'feature/f1' has been removed
        - You are now on branch 'develop'

        feature/f1 分支的代码会被合并到 develop 里面，然后删除该分支，切换回 develop. 到此，新功能开发这个场景完毕。在 f1 功能开发中，如果 f1 未完成，同时功能 f2 要开始进行，也是可以的。
---
-   发布上线，代号 0.1

    1.git flow release start 0.1
        Switched to a new branch 'release/0.1'

        Summary of actions:
        - A new branch 'release/0.1' was created, based on 'develop'
        - You are now on branch 'release/0.1'

        Follow-up actions:
        - Bump the version number now!
        - Start committing last-minute fixes in preparing your release
        - When done, run:

        git flow release finish '0.1'

        git-flow 从 develop 分支创建一个新的分支 release/0.1，并切换到该分支下，接下来要做的就是修改版本号等发布操作。

    2.git flow release finish 0.1
        Switched to branch 'master'
        Merge made by the 'recursive' strategy.
        f1      |    1 +
        version |    1 +
        2 files changed, 2 insertions(+)
        create mode 100644 f1
        create mode 100644 version
        Switched to branch 'develop'
        Merge made by the 'recursive' strategy.
        version |    1 +
        1 file changed, 1 insertion(+)
        create mode 100644 version
        Deleted branch release/0.1 (was d77df80).

        Summary of actions:
        - Latest objects have been fetched from 'origin'
        - Release branch has been merged into 'master'
        - The release was tagged '0.1'
        - Release branch has been back-merged into 'develop'
        - Release branch 'release/0.1' has been deleted

        git-flow 会依次切换到 master develop 下合并 release/0.1 里的修改，然后用 git tag 的给当次发布打上 tag 0.1，可以通过 git tag 查看所有 tag：
---
-   紧急 bug 修正，代号 bug1

    1.git flow hotfix start bug1
        Switched to a new branch 'hotfix/bug1'

        Summary of actions:
        - A new branch 'hotfix/bug1' was created, based on 'master'
        - You are now on branch 'hotfix/bug1'

        Follow-up actions:
        - Bump the version number now!
        - Start committing your hot fixes
        - When done, run:

            git flow hotfix finish 'bug1'

        git-flow 从 master 分支创建一个新的分支 hotfix/bug1，并切换到该分支下。接下来要做的就是修复 bug.

    2.git flow hotfix finish 'bug1'
        Switched to branch 'master'
        Merge made by the 'recursive' strategy.
        f1 |    2 +-
        1 file changed, 1 insertion(+), 1 deletion(-)
        Switched to branch 'develop'
        Merge made by the 'recursive' strategy.
        f1 |    2 +-
        1 file changed, 1 insertion(+), 1 deletion(-)
        Deleted branch hotfix/bug1 (was aa3ca2e).

        Summary of actions:
        - Latest objects have been fetched from 'origin'
        - Hotfix branch has been merged into 'master'
        - The hotfix was tagged 'bug1'
        - Hotfix branch has been back-merged into 'develop'
        - Hotfix branch 'hotfix/bug1' has been deleted

        git-flow 会依次切换到 master develop 分支下合并 hotfix/bug1，然后删掉 hotfix/bug1。到此，hotfix 完成。
---
-   remote branch

    1.push 一個 feature branch 到遠端
        git flow feature publish some_awesome_feature
        或 git push origin feature/some_awesome_feature
    2.追蹤一個遠端的 branch
        git flow feature track some_awesome_feature
        或 git checkout -b feature/some_awesome_feature -t origin/feature/some_awesome_feature
    3.刪除遠端的 branch
        git push origin :feature/some_awesome_feature
