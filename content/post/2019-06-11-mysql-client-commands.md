+++
date = "2019-06-11T00:00:00Z"
tags = ["mysql"]
categories = ["database"]
title = "MySQL 命令行客户端指令"

+++

MySQL 的 GUI 客户端有非常多的选择，自带的，开源的，商业的……GUI 客户端比自带的命令行客户端 `mysql` 方便友好很多，但是在批量处理等长连接操作上经常会卡住。论起连接的稳定性，命令行程序优势非常大，而且可以和服务端在同一台机子上或者同一个网段内，没有太大的网络延迟。以前我一直觉得命令行输入输出的显示和处理都不太方便，所以比较少用。最近看到了 [MySQL 客户端指令篇](https://dev.mysql.com/doc/refman/8.0/en/mysql-commands.html). 发现只要用得好，使用命令行也十分方便。

```
mysql> help

List of all MySQL commands:
Note that all text commands must be first on line and end with ';'
?         (\?) Synonym for `help'.
clear     (\c) Clear the current input statement.
connect   (\r) Reconnect to the server. Optional arguments are db and host.
delimiter (\d) Set statement delimiter.
edit      (\e) Edit command with $EDITOR.
ego       (\G) Send command to mysql server, display result vertically.
exit      (\q) Exit mysql. Same as quit.
go        (\g) Send command to mysql server.
help      (\h) Display this help.
nopager   (\n) Disable pager, print to stdout.
notee     (\t) Don't write into outfile.
pager     (\P) Set PAGER [to_pager]. Print the query results via PAGER.
print     (\p) Print current command.
prompt    (\R) Change your mysql prompt.
quit      (\q) Quit mysql.
rehash    (\#) Rebuild completion hash.
source    (\.) Execute an SQL script file. Takes a file name as an argument.
status    (\s) Get status information from the server.
system    (\!) Execute a system shell command.
tee       (\T) Set outfile [to_outfile]. Append everything into given
               outfile.
use       (\u) Use another database. Takes database name as argument.
charset   (\C) Switch to another charset. Might be needed for processing
               binlog with multi-byte charsets.
warnings  (\W) Show warnings after every statement.
nowarning (\w) Don't show warnings after every statement.
resetconnection(\x) Clean session context.

For server side help, type 'help contents'
```

上面是文档列出的所有命令。使用方法是括号里的字符。比如要退出，就 `\q`，然后 Enter.

输入的时候，可以将 `EDITOR` 设置成 `emacs`, 然后调用 `edit`, 然后就可以发挥想象力了。也可以 `source` 编辑好的命令。

结果显示增强最重要的就是 `ego` 了。实际数据很多表的列数非常多，终端的宽度根本不够用。`ego` 用竖排，效果有点像 JSON 数据。还可以设置好分页程序。

```
mysql> select * from information_schema.innodb_trx\G
*************************** 1. row ***************************
                    trx_id: 421187727341376
                 trx_state: RUNNING
               trx_started: 2019-06-10 21:24:32
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 0
       trx_mysql_thread_id: 9
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 0
          trx_lock_structs: 0
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 0
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
1 row in set (0.00 sec)
```

最后一个也很重要的是 `tee`. 怎么在交互的程序中导出结果，就用这个了。也可以导出到已经打开的文件中，类似于 `tail -f` 的效果。

PS: 上面的大部分指令在 `mycli` 上也是适用的。
