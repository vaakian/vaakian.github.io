---
title: 'Nginx Reverse Proxy Path'
date: 2022-04-12T15:16:09+08:00
draft: false
categories: ['Nginx']
---

假设服务器域名为`vaakian.com`，访问`uri`为`http://vaakian.com/api/user`，对比以下反向代理的效果。

## 1. 结尾都不打斜杠`/`

```nginx
location /api {
    proxy_pass http://127.0.0.1:8080
}
```

反代结果

```text
http://127.0.0.1:8080/api/user
```

## 2. location 打斜杠，proxy_pass 不打

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080
}
```

反代结果

```text
http://127.0.0.1:8080/api/user
```

所以，如果 proxy_pass 不打斜杠，那么实际访问的路由部分是原封不动的拼接的。

## 2. location 不打，proxy_pass 打斜杠

```nginx
location /api {
    proxy_pass http://127.0.0.1:8080/
}
```

反代结果，

```text
http://127.0.0.1:8080//user
```

## 2. location 和 proxy_pass 都打斜杠

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/
}
```

反代结果，

```text
http://127.0.0.1:8080/user
```

所以，proxy_pass 打斜杠时实际匹配到的 location 会被清楚掉。
