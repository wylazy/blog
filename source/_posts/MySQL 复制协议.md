---
id: 1534731278
title: MySQL 复制协议
date: 2018-08-20 10:14:38
categories: MySQL
mathjax2: false
---
### 概述
这里只介绍MySQL 的异步复制协议（MySQL 5.5增加了半同步复制功能，感兴趣的同学可以自己研究）。

MySQL的主从复制工作模式大致为，主库将执行的语句写入Binlog，由Dump线程将Binlog发送到从库的IO线程，IO线程将日志保存为Relay-Log，再由从库的SQL线程重放执行。本文主要研究主库Dump线程将Binlog发送到从库IO线程所使用的协议。

### 通信流程
首先从库向主库发起TCP连接，当连接建立完成后，主库向从库发送第一条数据包（Initial Handshake Packet），包含协议版本，服务器的版本，flag，以及认证相关信息。然后从库将用户名和加密的密码等认证信息发送给服务器。这一步的认证过程和普通MySQL客户端登录到MySQL服务器没什么区别。

然后从库向主库发送若干SELECT语句，获取服务器的时间戳，以及服务器版本等信息。

接着从库向主库发送一条COM_BINLOG_DUMP命令，开始复制过程。

当主库收到DUMP命令后，将Binlog中的Event一个接一个的发送到从库。

<img src='/images/1397025016.21.mysql_conn_protocol.png' />

### MySQL 一般数据包
每个MySQL数据包都是由的Header和Payload组成（包括Initial Handshake Packet在内的所有数据包）。Header由4个字节组成，3个字节的长度标识（FixedLengthInteger <http://dev.mysql.com/doc/internals/en/integer.html#packet-Protocol::FixedLengthInteger>），1个字节的序号。Playoad长度由Header部分指定。
<img src='/images/1397025142.11.mysql_packet.png' />

### 握手包
当从库与主库建立TCP连接后，主库向从库发送第一个数据包，包含了服务器的版本，以及服务器的能力，格式如下：
<img src='/images/1397629175.59.init.png' />

### DUMP命令
当从库连接到主库以后，从库向主库发送一条DUMP命令，开始复制过程。DUMP命令的结构如下图所示，第一字节的命令标识为DUMP命令的标识12。
<img src='/images/1397025217.19.dump_command.png' />
随后，主库会将Binlog的Event一个接一个的发送给从库，第一个为Rotate Event，这个Event包含了下一个Event对应的binlog的文件名称。第二个Event为Format Description Event，这个Event包含了所有Event的描述信息。MySQL协议里，在每个Event之前，都会有一个字节的00字段，这个字节在MySQL协议里叫做OK-Byte。
参考：<http://dev.mysql.com/doc/internals/en/com-binlog-dump.html>

### Event头字段
主库向从库发送的Event数据包也由Event头和Event消息体组成。从MySQL4.0开始，Event 头的长度固定为19字节。Event头的结构如下图所示：
<img src='/images/1419938843.52.event_header.png' />

### Rotate Event消息
Rotate Event比较简单，只包含一个Post Header和下一个Binlog的文件名称
<img src='/images/1397025336.27.rotate.png' />

### Format Description Event消息
FDE消息包含2个字节的Binlog版本，在MySQL5.0以后，Binlog版本是4；50字节的MySQL服务器版本，如果版本长度不足50，则后面补零；MySQL的Binlog支持26中Event，每种Event还可能有字节的Header，在FDE消息的最后，包含了各种Event对应的Event Header长度，每种Event Header长度对应一个字节。
<img src='/images/1397025391.31.fde.png' />

### Query Event消息
QueryEvent消息如上图所示，其中从“slave_proxy_id”到“2字节的状态变量长度”之间为Query Event的Header，共占13字节（对应FDE消息中的描述）。剩余部分为Payload，其中状态变量和Schema的长度由Header所指定。然后是一个字节的“00”，最后是SQL查询语句。
<img src='/images/1397025470.45.query.png' />

### 其他Event
请参考：<http://dev.mysql.com/doc/internals/en/binlog-event.html>

### 举例分析
场景为，主库在数据库db上的InnoDB数据表t上执行SQL 语句insert intot(val) values(‘a’)，其中在表t上带有自增主键。然后从库连接主库，开始复制过程。
<b>DUMP请求</b>
从库向主库发起DUMP命令
<img src='/images/1397025645.82.dump.png' />
<b>前4个字节是MySQL数据包的头：</b>

第1~3个字节：1b 00 00 为本数据包payload部分的长度27

第4个字节：00为包的序号

<b>剩下的27个字节（从第5字节到第31字节）是DUMP命令部分：</b>

第5个字节：12为COM_BINLOG_DUMP命令的标识

第6~9个字节：6a 00 00 00 为Binlog开始的位置

第10~11个字节：00 00 为两个字节的Flags

第12~15个字节：02 00 00 00 为从库的Server-id

第16~31个字节：Binlog的文件名

<b>主库回复Binlog</b>

主库收到DUMP命令后，向从库发送Event信息：
<img src='/images/1397025752.58.binlog_res.png' />
如上图，MySQL主库回复了6个部分的数据包，分别是6个Event：

Rotate Event

Format Description Event

Query Event: BEGIN

Intvar Event: #因为表t上带有自增主键，所以通过额外的Intval Event来保证主从在自增主键上的一致性

Query Event: insert into t(val) values(‘a’)

Xid Event: COMMIT

### Event标识对照表
参考：<http://dev.mysql.com/doc/internals/en/binlog-event-type.html>

Hex  | Event Name
---- | ----
0x00 | UNKNOWN_EVENT
0x01 | START_EVENT_V3
0x02 | QUERY_EVENT
0x03 | STOP_EVENT
0x04 | ROTATE_EVENT
0x05 | INTVAR_EVENT
0x06 | LOAD_EVENT
0x07 | SLAVE_EVENT
0x08 | CREATE_FILE_EVENT
0x09 | APPEND_BLOCK_EVENT
0x0a | EXEC_LOAD_EVENT
0x0b | DELETE_FILE_EVENT
0x0c | NEW_LOAD_EVENT
0x0d | RAND_EVENT
0x0e | USER_VAR_EVENT
0x0f | FORMAT_DESCRIPTION_EVENT
0x10 | XID_EVENT
0x11 | BEGIN_LOAD_QUERY_EVENT
0x12 | EXECUTE_LOAD_QUERY_EVENT
0x13 | TABLE_MAP_EVENT
0x14 | WRITE_ROWS_EVENTv0
0x15 | UPDATE_ROWS_EVENTv0
0x16 | DELETE_ROWS_EVENTv0
0x17 | WRITE_ROWS_EVENTv1
0x18 | UPDATE_ROWS_EVENTv1
0x19 | DELETE_ROWS_EVENTv1
0x1a | INCIDENT_EVENT
0x1b | HEARTBEAT_EVENT
0x1c | IGNORABLE_EVENT
0x1d | ROWS_QUERY_EVENT
0x1e | WRITE_ROWS_EVENTv2
0x1f | UPDATE_ROWS_EVENTv2
0x20 | DELETE_ROWS_EVENTv2
0x21 | GTID_EVENT
0x22 | ANONYMOUS_GTID_EVENT
0x23 | PREVIOUS_GTIDS_EVENT
