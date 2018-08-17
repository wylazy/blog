---
id: 1534471502
title: 轻量级网络抓包工具 ngrep
date: 2018-08-17 10:05:02
categories: Linux
tags:
  Linux
mathjax2: false
---
ngrep（network grep）是一款轻量级的网络抓包工具，是一个开源项目，代码托管在SourceForge上。ngrep依赖pcap库和GNU regex库，主要通过命令行接口使用。可以使用源码编译安装，在ubuntu上可以直接使用apt-get安装。

ngrep比tcpdump功能更强的的地方是，从名字也可以看出来，ngrep可以通过正则表达式过滤网卡的数据包，只显示这些匹配的结果，而且配置也很简单。

在ununtu上的安装方式如下：
```bash
sudo apt-get install ngrep
```

我们来看一下ngrep有哪些选项:
```bash
root@glz:~# ngrep -h
usage: ngrep <-hNXViwqpevxlDtTRM> <-IO pcap_dump> <-n num> <-d dev> <-A num>
             <-s snaplen> <-S limitlen> <-W normal|byline|single|none> <-c cols>
             <-P char> <-F file> <match expression> <bpf filter>
   -h  is help/usage  #打印帮助
   -V  is version information #显示版本
   -q  is be quiet (don't print packet reception hash marks) #只输出匹配的数据包
   -e  is show empty packets #空包也显示
   -i  is ignore case #不区分大小写匹配
   -v  is invert match #反向匹配，即只显示不匹配的数据包
   -R  is don't do privilege revocation logic #
   -x  is print in alternate hexdump format #用十六进制输出，和-W不能同时指定
   -X  is interpret match expression as hexadecimal #用十六进制表示正则表达式
   -w  is word-regex (expression must match as a word)
   -p  is don't go into promiscuous mode #不使用混杂模式
   -l  is make stdout line buffered 
   -D  is replay pcap_dumps with their recorded time intervals
   -t  is print timestamp every time a packet is matched  #在输出数据包之前，先输出当前时间
   -T  is print delta timestamp every time a packet is matched #是否输出数据包的间隔时间
   -M  is don't do multi-line match (do single-line match instead) #单行匹配
   -I  is read packet stream from pcap format file pcap_dump 
   -O  is dump matched packets in pcap format to pcap_dump
   -n  is look at only num packets
   -A  is dump num packets after a match #在匹配之后再暑促num个数据包
   -s  is set the bpf caplen
   -S  is set the limitlen on matched packets #在匹配时只搜索前面若干个字符
   -W  is set the dump format (normal, byline, single, none) #输出方式，byline比较有用，分行显示
   -c  is force the column width to the specified size
   -P  is set the non-printable display char to what is specified
   -F  is read the bpf filter from the specified file
   -N  is show sub protocol number
   -d  is use specified device instead of the pcap default #使用哪一个网卡，可以用ifconfig查看当前有哪些
   -K  is kill matching TCP connections
```

下面是一些抓包命令
```bash
ngrep -x -q -d lo port 3306 #抓取本机127.0.0.1上3306端口的数据包
ngrep -l -q -d eth0 "^GET |^POST " tcp and port 80 #抓取本机的HTTP GET和POST请求包
ngrep -l -q -d eth0 "" udp and port 53 #抓取UDP包
```

下面是MySQL在连接时抓获得数据包：
```plain
T 127.0.0.1:3306 -> 127.0.0.1:33290 [AP]
  45 00 00 00 0a 35 2e 31    2e 36 36 2d 30 75 62 75    E....5.1.66-0ubu
  6e 74 75 30 2e 31 30 2e    30 34 2e 31 00 a2 00 00    ntu0.10.04.1....
  00 5f 48 2a 36 70 55 37    50 00 ff f7 08 02 00 00    ._H*6pU7P.......
  00 00 00 00 00 00 00 00    00 00 00 00 6f 5e 69 72    ............o^ir
  24 57 6d 3d 28 4d 2e 3e    00                         $Wm=(M.>.       

T 127.0.0.1:33290 -> 127.0.0.1:3306 [AP]
  26 00 00 01 85 a6 03 00    00 00 00 01 08 00 00 00    &...............
  00 00 00 00 00 00 00 00    00 00 00 00 00 00 00 00    ................
  00 00 00 00 72 6f 6f 74    00 00                      ....root..      

T 127.0.0.1:3306 -> 127.0.0.1:33290 [AP]
  07 00 00 02 00 00 00 02    00 00 00                   ...........     

T 127.0.0.1:33290 -> 127.0.0.1:3306 [AP]
  21 00 00 00 03 73 65 6c    65 63 74 20 40 40 76 65    !....select @@ve
  72 73 69 6f 6e 5f 63 6f    6d 6d 65 6e 74 20 6c 69    rsion_comment li
  6d 69 74 20 31                                        mit 1           

T 127.0.0.1:3306 -> 127.0.0.1:33290 [AP]
  01 00 00 01 01 27 00 00    02 03 64 65 66 00 00 00    .....'....def...
  11 40 40 76 65 72 73 69    6f 6e 5f 63 6f 6d 6d 65    .@@version_comme
  6e 74 00 0c 08 00 08 00    00 00 fd 00 00 1f 00 00    nt..............
  05 00 00 03 fe 00 00 02    00 09 00 00 04 08 28 55    ..............(U
  62 75 6e 74 75 29 05 00    00 05 fe 00 00 02 00       buntu).........
```
