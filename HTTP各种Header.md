# HTTP各种Header
HTTP由RequestLine/StatusLine、Header、Body三个部分组成，Header是其中的重要组成部分。

本文着重介绍HTTP各种Header。

## Header几点注意
Header有几个需要注意的内容：
- 字段名不区分大小写，但是HTTP/2多了额外的限制，因为增加了头部压缩，要求在编码前必须转成小写
- 字段名不允许空格和下划线，可以使用连字符"-"，例如："Cache-Type"是合法的字段名，"Cache Type"、"Cache_Type"是不合法的
- 字段顺序没有意义，可以任意排列
- 大部分字段原则上不能重复，但是部分字段允许重复
- 部分字段要求内容是时间格式，HTTP的时间格式要求必须为GMT时区

## 各种Header详解

### Host
