---
id: 1534732376
title: MySQL 复制中对 Load Data的处理（译）
date: 2018-08-20 10:32:56
categories: MySQL
mathjax2: false
---
当MySQL执行LOAD DATA INFILE语句的时候，写binlog和其他语句有很大不同。一个LOAD DATA INFILE语句在binlog中可能变成了一个或者，若干个Event。这些Event记录了Load Data语句的附加信息，以及如何处理数据文件。

由于历史原因，一个LOAD DATA语句，可能对应4组不同的Event

1）在MySQL 3.23中，只有一个Event：Load_log_event（type code LOAD_EVENT=6）,Load_log_event只记录了文件名，没有记录文件本身。当Slave遇到Load_log_event时，Slave会再和Master建立一个连接，让Master把文件发送过来。这有一个缺点，就是binlog不是自包含的。如果在Master上文件已经被删除了，或者Slave连不上Master，文件就会传输失败。

2）在MySQL 4.0.0中，文件本身也会记录到binlog中。一个LOAD DATA INFILE语句会对应多个Event，Create_file_log_event (type code CREATE_FILE_EVENT = 8), Append_block_log_event (type code APPEND_BLOCK_EVENT = 9), Execute_load_log_event (type code EXEC_LOAD_EVENT = 10), and Delete_file_log_event (type code DELETE_FILE_EVENT = 11)，Event序列如下：

Create_file_log_event：传输1次

Append_block_log_event：传输0次，或者多次

Execute_load_log_event：传输1次（成功时）

Delete_file_log_event：传输1次（失败时）

Create_file_log_event也包含了LOAD DATA INFILE选项，这其实是一个设计上的缺陷。因为只有当遇到Execute_load_log_event时才能真正执行LOAD DATA语句。所以当Slave收到Create_file_log_event时，会把它写入临时文件，只有当遇到Execute_load_log_event时才从这个临时文件构造完整的LOAD DATA INFILE语句。

LOAD DATA语句在LOAD一个大文件的时候，会把一个大文件分块传，每个块一个Append_block_log_event。块大小不超过2^17 = 131072Bytes。

Create_file_log_event告诉Slave创建一个临时文件，并把文件的第一个块也写入临时文件（Create_file_log_event携带第一个文件块）。接下来会有若干Append_block_log_event，告诉Slave把它们追加写到这个临时文件。Execute_load_log_event告诉Slave把临时文件加载到数据表里面，或则是Delete_file_log_event告诉Slave不要加载临时文件，并把临时文件删除。当LOAD DATA语句在Master上执行失败时，Master会记录一条Delete_file_log_event。

3）MySQL 4.0.0新引入了NEW_LOAD_EVENT类型，typecode = 12。

NEW_LOAD_EVENT和以前的LOAD_EVENT差不多，不过支持了更长的分隔符。原始的LOAD_DATA_EVENT只用了一个字节表示分隔符（FIELDS TERMINATED BY）。后来在某个版本里，binlog格式支持了多字节作为分隔符，所以EVENT_TYPE也加入了支持。

4）MySQL 5.0.3里面又新添加了两个EVENT_TYPE。

Begin_load_query_log_event（typecode = 17）

Execute_load_query_log_event （typecode = 18）

一个LOAD DATA语句的Event序列可能如下：

Begin_load_query_log_event：传输1次

Append_block_log_event：传输0次，或者多次

Execute_load_log_event：传输1次（成功时）

Delete_file_log_event：传输1次（失败时）

在这个新序列中，Begin_load_query_log_event和Append_block_log_event几乎是相同的，Execute_load_log_event包含了LOAD DATA语句的文本。（而在4.0里面，LOAD DATA语句被记录在Create_file_log_event里面）。

这样就不在需要一个临时文件存放LOAD DATA INFILE的参数了，但是还是要有一个临时文件存放要被LOAD的数据。

示例部分请参照原文

原文地址：http://dev.mysql.com/doc/internals/en/load-data-infile-events.html

