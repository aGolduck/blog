+++
title = "Powershell style aliases for unix commands"
author = ["wenpin"]
date = 2020-06-12T17:55:00+08:00
lastmod = 2020-06-13T16:04:36+08:00
tags = ["unix", "Linux"]
categories = ["Linux"]
draft = false
+++

最近基本把 [Linux性能优化实战\_Linux\_性能调优-极客时间](https://time.geekbang.org/column/intro/140) 给看完了，从某种角度来讲，可以说就是学习了很多 unix/Linux 命令。我一直觉得 unix 的命令行命名风格是一个大败笔，使用大量的缩写，使得很多命令就算用了很久之后都不知道字面意思是什么，只能生背下来或者靠手熟， 尤其是非英语母语的人。很多时候在命令的 man page
都找不到全称是什么。比如说 `top`, 本来一直以为是显示最顶部的意思，结果竟然是 `table of processes`. 在计算机的上古时代显示器小，键盘难用，尚情有可原，但今天还有大量的新命令沿续这种风格，就是东施效颦。

如果是为了输入方便，可以做一下 `alias 'gst'='git status'` 一类的别名，完全没有必要最初起名时就起个完全不知道什么意思的新造词。在这方面，powershell 就做得非常好，不只表义性强，还都是 `Get-Commands`
这类的动宾组合。于是，我就学习一下 powershell，来次逆向 alias,
把日常用到的命令不明所以的还原一下原语义。以后不断更新。

但说实话，即使用了 `zsh-autosuggestions` 这类的工具，打完全称确实也比较累。日常使用还是原名（它本该是 alias)，不断地补充这个列表全当是为了方便记忆和加深理解吧。

```shell
# alias 'link'='ln' # link is used
alias 'advanced-top'='atop'
alias 'block-device-trace'='blktrace'
alias 'change-directory'='cd'
alias 'change-mode'='chmod'
alias 'change-owner'='chown'
alias 'chronos-table'='crontab'
alias 'client-url'='curl'
alias 'concatenate'='cat'
alias 'connection-track'='conntrack'
alias 'copy'='cp'
alias 'data-statistics'='dstat'
alias 'diagnostic-message'='dmesg'
alias 'disk-free'='df'
alias 'disk-usage'='du'
alias 'dns-interrogate'='dig'
alias 'dns-lookup'='nslookup'
alias 'execute'='exe'
alias 'execute-arguments'='xargs'
alias 'gnu-privacy-guard'='gpg'
alias 'interface-config'='ifconfig'
alias 'interface-top'='iftop'
alias 'internet-procotol'='ip'
alias 'interprocess-communication'='ipcs'
alias 'io-statistics'='iostat'
alias 'list'='ls'
alias 'list-block-devices'='lsblk'
alias 'list-dynamic-dependencies'='ldd'
alias 'list-open-files'='lsof'
alias 'make-directory'='mkdir'
alias 'manual'='man'
alias 'member-leak'='memleak'
alias 'move'='mv'
alias 'net-concatenate'='netcat'
alias 'net-top'='nethogs'
alias 'network-hardware-tool'='ethtool'
alias 'network-mapper'='nmap'
alias 'network-statistics'='netstat'
alias 'performance'='perf'
alias 'print-working-directory'='pwd'
alias 'privacy-guard'='gpg'
alias 'process-grep'='pgrep'
alias 'process-kill'='pkill'
alias 'process-memory-map'='pmap'
alias 'process-statistics'='pidstat'
alias 'process-status'='ps'
alias 'process-tree'='pstree'
alias 'processor-statistics'='mpstat'
alias 'read-executable-linking-file'='readelf'
alias 'remove'='rm'
alias 'search-regular'='grep'
alias 'secure-copy'='scp'
alias 'secure-shell'='ssh'
alias 'socket-statistics'='ss'
alias 'stream-editor'='sed'
alias 'substitude-user'='su'
alias 'system-activity-report'='sar'
alias 'system-call-trace'='strace'
alias 'table-of-processes'='top'
alias 'tape-archive'='tar'
alias 'traffic-control'='tc'
alias 'translate'='tr'
alias 'unique'='uniq'
alias 'unix-limit'='ulimit'
alias 'unix-name'='uname'
alias 'virtual-memory-statistics'='vmstat'
alias 'web-get'='wget'
alias 'word-count'='wc'
```
