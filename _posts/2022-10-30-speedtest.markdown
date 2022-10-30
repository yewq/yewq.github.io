---
layout: post
title:  "Speedtest"
date:   2022-10-30 19:54:51 +0800
categories: linux
---

用 Javascript 实现，轻量级的网络测速软件。

## 安装

```shell
$ git clone https://github.com/librespeed/speedtest
$ cd speedtest/
```

- 选择 `example-singleServer-gauges.html` 文件，样式为仪表盘，不保存结果。
- IP 地址解析使用 PHP 实现，将其注释。

```shell
$ git checkout -b 5.2.5
$ cp example-singleServer-gauges.html index.html
$ diff -Nur example-singleServer-gauges.html index.html
--- example-singleServer-gauges.html    2022-10-30 17:23:45.076507652 +0800
+++ index.html  2022-10-30 18:20:42.999684751 +0800
@@ -79,7 +79,6 @@
        if(!forced&&s.getState()!=3) return;
        if(uiData==null) return;
        var status=uiData.testState;
-       I("ip").textContent=uiData.clientIp;
        I("dlText").textContent=(status==1&&uiData.dlStatus==0)?"...":format(uiData.dlStatus);
        drawMeter(I("dlMeter"),mbpsToAmount(Number(uiData.dlStatus*(status==1?oscillate():1))),meterBk,dlColor,Number(uiData.dlProgress),progColor);
        I("ulText").textContent=(status==3&&uiData.ulStatus==0)?"...":format(uiData.ulStatus);
@@ -105,7 +104,6 @@
        I("ulText").textContent="";
        I("pingText").textContent="";
        I("jitText").textContent="";
-       I("ip").textContent="";
 }
 </script>
 <style type="text/css">
@@ -251,9 +249,6 @@
                                <div class="unit">Mbps</div>
                        </div>
                </div>
-               <div id="ipArea">
-                       <span id="ip"></span>
-               </div>
        </div>
        <a href="https://github.com/librespeed/speedtest">Source code</a>
 </div>
```

## Nginx 配置

```shell
$ sudo vim /etc/nginx/conf.d/speed.yewq.cn.conf 
server {
    listen 443 ssl;
    server_name speedtest.yewq.cn;

    ssl_certificate /home/yewq/yewq.cn/certs/speedtest.yewq.cn_bundle.crt;
    ssl_certificate_key /home/yewq/yewq.cn/certs/speedtest.yewq.cn.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;

    location / {
        root /home/yewq/yewq.cn/speedtest;
        index index.html;
    }
}

server {
    listen 80;
    server_name speedtest.yewq.cn;
    return 301 https://$host$request_uri;
}
```

```shell
$ sudo nginx -t
$ sudo systemctl reload nginx.service
``` 

## 运行

- [【主页】speedtest.yewq.cn](https://speedtest.yewq.cn/)

## 参考资料

1. [【源码】Speedtest](https://github.com/librespeed/speedtest)
