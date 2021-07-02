---
title: linux 命令统计日志常用方法
date: 2021-07-02
tags: [linux]
---

通过使用 linux 命令可以很方便地对日志进行统计和分析。当需要进行一些简单的文本处理、数据统计、或者服务有异常需要去排查日志的时候，掌握一种统计日志的技巧可以让工作事半功倍。

<!-- more -->

假设有一个包含下面内容的日志文件 access.log。我们以统计这个文件的日志为例。

```
date=2017-09-23 13:32:50 | ip=40.80.31.153 | method=GET | url=/api/foo/bar?params=something | status=200 | time=9.703 | bytes=129 | referrer="-" | user-agent="Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/535.7 (KHTML, like Gecko) Chrome/16.0.912.63 Safari/535.7" | cookie="-"
date=2017-09-23 00:00:00 | ip=100.109.222.3 | method=HEAD | url=/api/foo/healthcheck | status=200 | time=0.337 | bytes=10 | referrer="-" | user-agent="-" | cookie="-"
date=2017-09-23 13:32:50 | ip=40.80.31.153 | method=GET | url=/api/foo/bar?params=anything | status=200 | time=8.829 | bytes=466 | referrer="-" | user-agent="GuzzleHttp/6.2.0 curl/7.19.7 PHP/7.0.15" | cookie="-"
date=2017-09-23 13:32:50 | ip=40.80.31.153 | method=GET | url=/api/foo/bar?params=everything | status=200 | time=9.962 | bytes=129 | referrer="-" | user-agent="Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/535.7 (KHTML, like Gecko) Chrome/16.0.912.63 Safari/535.7" | cookie="-"
date=2017-09-23 13:32:50 | ip=40.80.31.153 | method=GET | url=/api/foo/bar?params=nothing | status=200 | time=11.822 | bytes=121 | referrer="-" | user-agent="Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/535.7 (KHTML, like Gecko) Chrome/16.0.912.63 Safari/535.7" | cookie="-"
```

不同的场景下对应的日志格式不一样，比如上面的日志是以如下规则记录的：

```
date | ip | method | url | status | time | bytes | referrer | user-agent | cookie
```

> 注意 mac 系统和 linux 系统中的命令行为可能不同，以下命令请在 linux 系统中使用

## 筛选日志

统计日志时，我们可能不关心 HEAD 请求，或者只关心 GET 请求，这里首先需要筛选日志，可以使用 grep 命令。`-v` 的含义是排除匹配的文本行。

```sh
grep GET access.log # 只统计 GET 请求
grep -v HEAD access.log # 不统计 HEAD 请求
grep -v 'HEAD\|POST' access.log # 不统计 HEAD 和 POST 请求
```

## 提取字段

### 查看接口耗时情况

我们可以将每行的 time 匹配出来，然后做一个排序。使用 awk 的 match 方法可以匹配正则：

```sh
awk '{ match($0, /time=([0-9]+\.[0-9]+)/, result); print result[1]}' access.log
```

awk 命令使用方法如下：

```sh
awk '{pattern + action}' {filenames}
```

我们实际上只用到了 action：`match($0, /time=([0-9]+\.[0-9]+)/, result); print result[1]` 这一段。

match 方法接收三个参数：需要匹配的文本、正则表达式、结果数组。`$0` 代表 awk 命令处理的每一行，结果数组是可选的，因为我们要拿到匹配结果所以这里传入了一个 result 数组，用来存储匹配后的结果。

注意这里的正则我没有使用 `\d` 来表示数字，因为 awk 指令默认使用 “EREs"，不支持 `\d` 的表示，具体请看 [linux shell 正则表达式 (BREs,EREs,PREs) 差异比较](http://www.cnblogs.com/chengmo/archive/2010/10/10/1847287.html)。

result 数组实际上和 javascript 里的结果数组很像了，所以我们打印出第二个元素，即匹配到的内容。执行完这行命令后结果如下：

```
9.703
0.337
8.829
9.962
11.822
```

当然实际上一天的日志可能是成千上万条，我们需要对日志进行排序，且只展示前 3 条。这里使用到 sort 命令。

sort 命令默认从小到大排序，且当作字符串排序。所以默认情况下使用 sort 命令之后 "11" 会排在 "8" 前面。那么需要使用 `-n` 指定按数字排序，`-r` 来按从大到小排序，然后我们查看前 3 条：

```sh
awk '{ match($0, /time=([0-9]+\.[0-9]+)/, result); print result[1]}' access.log | sort -rn | head -3
```

结果：

```
11.822
9.962
9.703
```

## 根据某字段排序

### 查看耗时最高的接口

当然我们一般不会只查看接口耗时情况，还需要把具体日志也打印出来，上面的命令就不能满足要求了。

awk 的打印默认是按空格分隔的，意思是 `2017-09-23 GET` 这一行如果使用 `awk '{print $1}'` 会打印出 "2017-09-23"，类似地，`$2` 会打印出 GET。

根据日志特征，我们可以使用 | 来作为分隔符，这样就能打印出各个我们感兴趣的值了。因为我们想找出耗时最高的接口，那么我们把 time、date 和 url 单独找出来。

awk 的 -F 参数用来自定义分隔符。然后我们可以数一下三个部分按 | 分隔后分别是第几个：time 是第 6 个、date 是第 1 个、url 是第 4 个。

```sh
awk -F '|' '{print $6 $1 $4}' access.log
```

这样打出来结果为：

```
 time=9.703 date=2017-09-23 13:32:50  url=/api/foo/bar?params=something
 time=0.337 date=2017-09-23 00:00:00  url=/api/foo/healthcheck
 time=8.829 date=2017-09-23 13:32:50  url=/api/foo/bar?params=anything
 time=9.962 date=2017-09-23 13:32:50  url=/api/foo/bar?params=everything
 time=11.822 date=2017-09-23 13:32:50  url=/api/foo/bar?params=nothing
```

因为我们想按 time 来排序，而 sort 可以按列来排序，而列是按空格分隔的，我们目前第一列是 time=xxx，是不能排序的，所以这里要想办法把 time= 给去掉，因为我们很鸡贼地把耗时放在了第一列，那么其实再通过 time= 进行分隔一下就行了。

```sh
awk -F '|' '{print $6 $1 $4}' access.log | awk -F 'time=' '{print $2}'
```

结果：

```
9.703 date=2017-09-23 13:32:50  url=/api/foo/bar?params=something
0.337 date=2017-09-23 00:00:00  url=/api/foo/healthcheck
8.829 date=2017-09-23 13:32:50  url=/api/foo/bar?params=anything
9.962 date=2017-09-23 13:32:50  url=/api/foo/bar?params=everything
11.822 date=2017-09-23 13:32:50  url=/api/foo/bar?params=nothing
```

使用 sort 的 `-k` 参数可以指定要排序的列，这里是第 1 列；再结合上面的排序，就能把耗时最高的日志打印出来了：

```sh
awk -F '|' '{print $6 $1 $4}' access.log | awk -F 'time=' '{print $2}' | sort -k1nr | head -3
```

结果：

```
11.822 date=2017-09-23 13:32:50  url=/api/foo/bar?params=nothing
9.962 date=2017-09-23 13:32:50  url=/api/foo/bar?params=everything
9.703 date=2017-09-23 13:32:50  url=/api/foo/bar?params=something
```

## 统计某字段条数、去重条数

### 统计请求次数最多的接口

如果需要统计哪些接口每天请求量是最多的，只需要新引入 `uniq` 命令。

我们已经可以通过 `grep -v HEAD access.log | awk -F '|' '{print $4}'` 来筛选出所有的 url，uniq 命令可以删除 相邻 的相同的行，而 `-c` 可以输出每行出现的次数。

所以我们先把 url 排序以让相同的 url 放在一起，然后使用 `uniq -c` 来统计出现的次数：

```sh
grep -v HEAD access.log | awk -F '|' '{print $4}' | sort  | uniq -c
```

因为示例日志数量太少，我们假设日志里有多条，那么结果应该类似下面：

```
1  url=/api/foo/bar?params=anything
19  url=/api/foo/bar?params=everything
4  url=/api/foo/bar?params=nothing
5  url=/api/foo/bar?params=something
```

接下来再 sort 即可：

```sh
grep -v HEAD access.log | awk -F '|' '{print $4}' | sort  | uniq -c | sort -k1nr | head -10
```
