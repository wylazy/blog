---
id: 1534732562
title: 编写 MySQL 插件
date: 2018-08-20 10:36:02
categories: MySQL
mathjax2: false
---
众所周知，MySQL是支持插件式存储引擎的，意思是MySQL源码中开放了存储引擎相关API，只要插件实现相关的API，就能安装到MySQL中，并作为存储引擎开始工作了。其实MySQL支持多种类型的插件，比如UDF，Daemon，认证，半同步，存储引擎。其中UDF和Daemon插件都非常简单，UDF只是实现了一个在MySQL的SQL接口里可以调用的函数；而Daemon插件只要按照MySQL Plugin的声明方式，声明plugin的描述以及入口函数，从这个plugin入口函数开始，你可以编写任意的代码，比如创建一个后台线程，甚至直接调用MySQL server的源码，访问数据表的数据。

本文主要描述在MySQL中编写Daemon plugin的方法，本文创建一个为monitor的后台plugin，后台每隔若干秒将thread_count等打印到日志中。具体代码如下：

声明MySQL插件的描述以及入口出口函数
```cpp
struct st_mysql_daemon monitor_info = { MYSQL_DAEMON_INTERFACE_VERSION};
 
mysql_declare_plugin(monitor_plugin)
{
    MYSQL_DAEMON_PLUGIN,
    &monitor_info,
    "monitoring",
    "wylazy",
    "monitoring mysql thread",
    PLUGIN_LICENSE_BSD,
    monitoring_plugin_init,
    monitoring_plugin_deinit,
    0x0100, //1.0
    NULL,
    vars_system_var,
    NULL
}
mysql_declare_plugin_end;
 
/**
 * 具体的含义如下：
 *
 * struct st_mysql_plugin {
 *   int type;                  // 插件类型, 比如MYSQL_DAEMON_PLUGIN,MYSQL_STORAGE_ENGINE_PLUGIN
 *   void * info;
 *   const char * name;         // INSTALL PLUGIN用到的插件名字
 *   const char * author;       // 插件的作者
 *   const char * descr;        // 插件的描述
 *   int license;               // PLUGIN_LICENSE_GPL, PLUGIN_LICENSE_BSD
 *   int (* init)(void *);      // 插件的入口函数
 *   int (* deinit)(void *);    // 插件的退出函数
 *   unsigned int version;      // 低8个bit存minor version，其他的为major version
 *   struct st_mysql_show_var * status_vars; // 通过SHOW STATUS显示的状态信息
 *   struct st_mysql_sys_var ** system_vars; // 通过SHOW VARIABLES显示的变量信息
 *   void * __reserved;         // 在MySQL 5.1里未用到
 * }
 *
 */
```
其中在结构体st_mysql_plugin中，成员name是插件的名称，(* init)(void *)是插件的入口函数，(* deinit)(void *)是插件的退出函数。
```bash
install plugin monitoring so name xxxxx.so
```
当通过mysql命令行执行如上命令的时候，MySQL的执行过程大概为：通过插件的名称'monitoring'找到相应的插件，然后执行对应的init()函数。在uninstall plugin的时候执行deinit()函数。

因为我们的plugin想在后台执行，所以需要在init中创建一个后台执行的线程。具体代码如下：
```cpp
//后台线程
pthread_handler_t monitoring(void * p) {
    char buffer[MONITOR_BUFFER];
    char time_str[20];
 
    while (1) {
        struct timeval tv;
        tv.tv_sec = waiting_seconds;
        tv.tv_usec = 0;
         
        //每隔若干秒，打印一次日志
        if (select(1, NULL, NULL, NULL, &tv) == 0) {
            get_date(time_str, GETDATE_DATE_TIME, 0);
            sprintf(buffer, "%s: %u of %lu clients connected, %lu connections made\n", 
                    time_str, thread_count, max_connections, thread_id);
            write(monitoring_file, buffer, strlen(buffer));
        } else {
            fprintf(stderr, "select return error");
            break;
        }
    }
}
 
/**
* plugin的入口
* 返回0：安装插件成功
* 返回非0：安装插件失败。mysqld内部会调用deinit()方法做清理
*/
static int monitoring_plugin_init(void *p) {
    pthread_attr_t attr;
    char monitoring_filename[FN_REFLEN];
    char buffer[MONITOR_BUFFER];
    char time_str[20];
 
    //打开日志文件
    fn_format(monitoring_filename, "monitor", "", ".log", MY_REPLACE_EXT | MY_UNPACK_FILENAME);
    unlink(monitoring_filename);
    if ((monitoring_file = open(monitoring_filename, O_CREAT|O_RDWR, 0644)) < 0) {
        fprintf(stderr, "plugin 'monitoring' could not create file %s", monitoring_filename);
        return 1;
    }
 
    get_date(time_str, GETDATE_DATE_TIME, 0);
    sprintf(buffer, "Monitoring started at %s\n", time_str);
    write(monitoring_file, buffer, strlen(buffer));
 
    //创建后台线程
    if (pthread_create(&monitoring_thread, NULL, monitoring, NULL) != 0) {
        fprintf(stderr, "Plugin monitoring could not create monitoring thread\n");
        return 1;
    }
    return 0;
}
 
//plugin的出口
static int monitoring_plugin_deinit(void * p) {
    char buffer[MONITOR_BUFFER], time_str[20];
     
    //结束后台线程
    pthread_cancel(monitoring_thread);
    pthread_join(monitoring_thread, NULL);
 
    get_date(time_str, GETDATE_DATE_TIME, 0);
    sprintf(buffer, "Monitoring stopped at %s\n", time_str);
    write(monitoring_file, buffer, strlen(buffer));
 
    //关闭日志文件
    close(monitoring_file);
    return 0;
}
```
在plugin启动的时候，我们在mysql的数据目录下创建了一个日志文件monitor.log，同时创建了一个后台线程打印日志，在卸载plugin的时候，停止后台线程，并关闭日志文件。

为了可以动态的控制打印日志的时间间隔，我们还在plugin中添加了一个动态的变量waiting_seconds，控制后台线程打印日志的时间间隔。
```cpp
int waiting_seconds = 0;
 
//将变量与waiting_seconds 关联
static MYSQL_SYSVAR_INT(mo_waiting_seconds, waiting_seconds, PLUGIN_VAR_RQCMDARG, 
                       "waiting time before print log", NULL, NULL
                       , 5  //默认值
                       , 0  //最小值
                       , 600 //最大值
                       , 0);
                        
//将mo_waiting_seconds 添加到系统中                      
struct st_mysql_sys_var * vars_system_var[] =
{
    MYSQL_SYSVAR(mo_waiting_seconds)
    , NULL
};
```
按照如上的方式，即可在mysql客户端中通过如下命令动态改变打印日志的时间间隔：
```sql
set global monitoring_mo_waiting_seconds = 8;
```
编译plugin的时候还需要依赖MySQL，既可以通过依赖MySQL源代码的方式编译，又可以通过依赖mysql库的方式编译。通过这两种方式编译所支持的功能是不同的，如果直接依赖mysql源码，就可以包含mysql在sql目录下的头文件，使用THD等服务器的数据类型，甚至是访问mysqld的全局变量，调用mysqld的函数等。而依赖mysql库编译的时候，功能就相对弱一些。这里为了方便起见，选择通过依赖mysql库的方式编译，Makefile的内容如下：
```bash
CC = gcc
MYSQL_PATH = $(HOME)/programs/mysql
MYSQL_CONFIG = $(MYSQL_PATH)/bin/mysql_config
 
INCLUDES = ${shell $(MYSQL_CONFIG) --include}
# LIBS = ${shell $(MYSQL_CONFIG) --libs}
 
CPPFLAGS := -g $(CPPFLAGS) $(INCLUDES) -fPIC -DMYSQL_DYNAMIC_PLUGIN
 
SO_BASE=monitor
 
all: $(SO_BASE).so
 
$(SO_BASE).so: $(SO_BASE).o
        $(CC) -o $@ -shared $<
 
install: all
        cp $(SO_BASE).so $(MYSQL_PATH)/lib/mysql/plugin/
 
clean:
        rm -f *.o *.gch *.so
```

相关说明：

1. 同一个名称的plugin只能被install一次，如果install多次，只有第一次可以install成功，后面的都会失败，而且连plugin的init方法都不会调用
2. 在install plugin以后，如果没有卸载plugin。如果将mysqld重启，plugin会在重启的时候自动install





