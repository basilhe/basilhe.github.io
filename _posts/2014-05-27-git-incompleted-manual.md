---
layout: post
title: "Git 不完全手册"
description: "收集整理 Git 相关手册和技巧"
category: devops 
tags: [git,scm,manual]
---
{% include JB/setup %}



## Help
  * some concept guides: @git help -g@ 
  * available subcommands: @git help -a@ 

### Undo git reset

<pre>
git reset HEAD~
git reset HEAD@{1}

# Git keeps a log of all ref updates (e.g., checkout, reset, commit, merge). You can view it by typing:
git reflog
</pre>

## References

### Manuals
  * [Git User's Manual - Kernel.org](https://www.kernel.org/pub/software/scm/git/docs/user-manual.html)
  * [Git Tutorials - Atlassian](https://www.atlassian.com/git/tutorial)
  * [Git命令参考手册](http://www.oschina.net/question/156344_148084)

### Workflows
  * [Git Workflows - Atlassian](https://www.atlassian.com/git/workflows)
  * [A Successful Git Branching Model](http://nvie.com/posts/a-successful-git-branching-model/) with Tool [git-flow](https://github.com/nvie/gitflow)
  * [Git-Flow Cheat Sheet](http://danielkummer.github.io/git-flow-cheatsheet/)
  * [Stop using "git pull". Be polite](https://github.com/aanand/git-up)

### Issues
  * [Revert a commit already pushed to a remote repository](http://christoph.ruegg.name/blog/git-howto-revert-a-commit-already-pushed-to-a-remote-reposit.html)

### Cool Shells
  * [Oh my zsh](https://github.com/robbyrussell/oh-my-zsh)
  * [Why Zsh is cooler than your shell](http://www.slideshare.net/jaguardesignstudio/why-zsh-is-cooler-than-your-shell-16194692)

### Snippets
 * 删除不在 git 仓库中的其余文件
<pre>
git clean -fxd {path}
</pre>
