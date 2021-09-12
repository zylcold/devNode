从提交历史或者工作目录中查找一个字符串或者正则表达式


```shell
$ git grep -n gmtime_r

compat/gmtime.c:3:#undef gmtime_r 
compat/gmtime.c:8:      return git_gmtime_r(timep, &result); 
compat/gmtime.c:11:struct tm *git_gmtime_r(const time_t *timep, struct tm *result)

```

传入 `-n`  参数来输出 Git 所找到的匹配行行号


```shell
$ git grep --count 
gmtime_r compat/gmtime.c:4 
compat/mingw.c:1 
compat/mingw.h:1
```
使用  --count  选项来使 Git 输出概述的信息



```shell
$ git grep -p gmtime_r *.c 
date.c=static int match_multi_number(unsigned long num, char c, const char *date, char  *end, struct tm *tm) 
date.c:         if (gmtime_r(&now, &now_tm))
```

传入 `-p`  选项 ,看匹配的行是属于哪一个方法或者函数

