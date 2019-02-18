---
title: mysql embedded server example
date: 2019-02-18 22:33:28
tags:
---

MySQL有一种作为嵌入运行服务器的[方式](https://dev.mysql.com/doc/refman/5.7/en/libmysqld.html)。此处做个记录。
另外5.7的[mysql\_library\_init()](https://dev.mysql.com/doc/refman/5.7/en/mysql-library-init.html)还有相关的说明，[8.0的](https://dev.mysql.com/doc/refman/8.0/en/mysql-library-init.html)已经没有了。

<!-- more -->

```c
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <mysql/mysql.h>

int main(int argc, char *argv[])
{
    char *mysql_init_args[] = {
        "mysql_embedded_test",
        "--datadir=/tmp/mysql",
        "--default-storage-engine=InnoDB",
        "--skip-grant-tables",
        "--myisam-recover-options=FORCE",
        "--key-buffer-size=16777216",
        "--character-set-server=utf8mb4",
        "--collation-server=utf8mb4_bin",
    };

    if (mysql_library_init(sizeof(mysql_init_args) / sizeof(mysql_init_args[0]), mysql_init_args, NULL) != 0) {
        fprintf(stderr, "mysql library init error\n");
        exit(EXIT_FAILURE);
    }

    MYSQL *m = mysql_init(NULL);

    if (mysql_options(m, MYSQL_OPT_USE_EMBEDDED_CONNECTION, NULL) != 0) {
        fprintf(stderr, "mysql set options error\n");
    }

    if (!mysql_real_connect(m, NULL, NULL, NULL, NULL, 0, NULL, 0)) {
        fprintf(stderr, "mysql connect error\n");
    }

    if (mysql_query(m, "create database mysql_embedded_test;") != 0) {
        fprintf(stderr, "create database failed: %s\n", mysql_error(m));
    }

    if (mysql_query(m, "use mysql_embedded_test;") != 0) {
        fprintf(stderr, "select database failed: %s\n", mysql_error(m));
    }

    if (mysql_query(m, "create table mysql_embedded_test_table (\
        id bigint(20) unsigned auto_increment,\
        name varchar(255), primary key (id));") != 0) {
        fprintf(stderr, "create table failed: %s\n", mysql_error(m));
    }

    if (mysql_query(m, "insert into mysql_embedded_test_table (name) values ('aaaa');") != 0) {
        fprintf(stderr, "insert table failed: %s\n", mysql_error(m));
    }

    if (mysql_query(m, "select * from mysql_embedded_test_table;") != 0) {
        fprintf(stderr, "select table failed: %s\n", mysql_error(m));
    }

    MYSQL_RES *result = mysql_store_result(m);
    MYSQL_ROW row;

    while (row = mysql_fetch_row(result)) {
        printf("%s - %s\n", row[0], row[1]);
    }

    mysql_free_result(result);
    mysql_close(m);
    mysql_library_end();

    return EXIT_SUCCESS;
}
```

当前编译的时候需要链接libmysqld.so，而不是libmysqlclient.so。另外运行时需要保证数据目录（比如这里是/tmp/mysql）已存在。
