+++
date = "2020-02-29T00:00:00Z"
tags = ["zsh", "oh-my-zsh"]
categories = ["unix"]
title = "oh-my-zsh gnu_utils 升级导致提示符路径名消失"

+++

记录下使用 `oh-my-zsh` 踩坑的过程。

今天被自己的 zsh 环境坑到了。我用 antigen 来管理 zsh 插件，插件不多，主要是 oh-my-zsh 少量几个和 zsh-users 的仿 fish 插件。用 z 插件跳转到了一个同名的文件夹里，然后在编辑器使劲修改都没有用。花了几个小时，一直找不出问题。重装依赖包在编辑器里发现依赖包文件夹怎么也刷不出来，就觉得诡异。跑到 zsh 下 pwd, 天，居然路径不一致。

以前用 oh-my-zsh 直接管理插件的时候用的是默认的主题，后来改用 antigen 就没有设置。心想，这真是懒惰的罪过啊。我只是没有配置显示路径而已啊。开始搜有什么合适的主题，都花里糊哨的，不喜欢。这不就是一个提示符号的配置吗，修改下 PS1 变量就行了，不用那么复杂。

回到家里，准备用家里的电脑配置，却发现怎么这有全路径显示啊。想起来以前我还嫌全路径太长，原来一直是有的才对啊。公司和家里的电脑的配置一模一样的。于是在 .zshrc 里二分法加 echo $PS1. antigen apply 之前和公司一样都是 %m%#. antigen apply 以后，家里的电脑就变成了 %n@%m:%~%#，有全路径配置了，公司的电脑还是 %m%#. 果然是插件在作妖。

用的插件也一模一样。那就是版本的问题。发现不同 zsh 版本都一样，于是推测是插件的问题。前几天公司的电脑 antigen update 了一波。于是我用家里的电脑也 antigen update 一波。果然，家里的电脑也没有全路径提示了。

本来到这也可以结束了，在 .zshrc antigen apply 后 export 一个喜欢的 PS1 就行了。但是还是比较好奇，到底是哪里引起的问题。可是被我 antigen update 了之后车祸现场消失了，怎么办。

查了下，发现 antigen 有 revert 功能，于是回滚，再用二分调试大法，找到了是由于 oh-my-zsh 的 gnu_utils 升级引起的。

假如 antigen 没有 revert 功能怎么办。现在很多解释型语言的模块管理器采用了直接 clone/pull git 仓库的办法，antigen 也是。在 ~/.antigen/bundles 里可以找到 antigen 管理的插件的本地仓库。用 git reflog 可以找到上次 HEAD 的 commit. 当然不像 antigen revert 一样把所有的插件回滚，得费一些劲。
