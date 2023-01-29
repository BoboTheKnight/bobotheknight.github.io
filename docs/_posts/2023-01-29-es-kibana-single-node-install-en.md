---
layout: post
title: "Single Node ElasticSearch and Kibana Installation Instructions"
date: 2023-01-29 00:10:10 -0500
categories: ops
---

[简体中文](https://bobotheknight.github.io/docs/ops/2021/12/02/es-kibana-single-node-install-zh.html)

## Summary
&ensp;&ensp;&ensp;&ensp;To support the new features, we have added ES nodes. You can configure ES as a single node or cluster depending on the data situation and status; enable Xpack, enable permission authentication (requires Kibana installation).
The official documentation Set up Elasticsearch has installation instructions for each OS, [Installing Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/install-elasticsearch.html) page provides links to instructions for various installation packages, which can be consulted. This document provides instructions for installing ES and Kibana on a single-node Linux server.
---
## Setup ES
1. First make sure you have a JDK in your environment. Elasticsearch 7.x package includes the OpenJDK11 package, if you need to use your own version, refer to the [official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html) to set the JAVA_HOME environment variable.

2. Create an ES-specific user, as it is not possible to start with *root* user.
    ```
    useradd elasticsearch
    ```
   Create the ES directory and grant permissions to the *elasticsearch* user.
   ```
   cd /home
   mkdir /es
   chown -R elasticsearch:elasticsearch /home/es/
   ```

3. Upload **elasticsearch-7.14.0-linux-x86_64.tar.gz** in the zip archive, or you can go to the [official website](https://www.elastic.co/cn/downloads/elasticsearch) to download it or use other methods to download it.

4. Unpacked
   ```
   tar -zxvf elasticsearch-7.14.0-linux-x86_64.tar.gz
   ```

5. Modify the configuration file and go to the unpacked directory
   ``` 
   cd elasticsearch-7.14.0/config
   ```
   First backup the configuration file **elasticsearch.yml**, and then modify
   ````
   cp elasticsearch.yml elasticsearch.yml.bak 
   vim elasticsearch.yml
   ------------------------------
   network.host: ${ip of this server}
   http.port: 9200
   discovery.seed_hosts: ["${ip of this server}"]
   discovery.type: single-node # single-node mode
   ------------------------------
   ````
   
6. Open ports of 9200 and 9300, or turn off the firewall.

7. 根据配置文件，创建data目录存储es数据
   ```
   mkdir data
   ```
   
8. 给ES用户所有ES相关的权限；切换到*elasticsearch*用户；在bin目录下启动ES
    ``` 
    chown -R elasticsearch:elasticsearch ./*
    su – elasticsearch
    cd /home/es/elasticsearch-7.14.0/bin
    ./elasticsearch &   
    ```
    启动后可能会出现报错：
    ```
    trying to update state on non-existing task geoip-downloader
    [2021-08-11T15:33:57,318][ERROR][o.e.i.g.GeoIpDownloader ] [18789989a729] error updating geoip database [GeoLite2-Country.mmdb]
    ```
    
    该报错可以忽略；若想解决请看[附录 1.解决ES启动报错问题](#附录)

9. 验证启动情况，在本机执行
   ```
   curl http://${ip}:9200/
   ```
   得到返回结果如:
   ```json
   {
     "name" : "localhost.localdomain",
     "cluster_name" : "oss-es",
     "cluster_uuid" : "HKCnF4l_TOSW8-mznxM5eg",
     "version" : {
       "number" : "7.14.0",
       "build_flavor" : "default",
       "build_type" : "tar",
       "build_hash" : "dd5a0a2acaa2045ff9624f3729fc8a6f40835aa1",
       "build_date" : "2021-07-29T20:49:32.864135063Z",
       "build_snapshot" : false,
       "lucene_version" : "8.9.0",
       "minimum_wire_compatibility_version" : "6.8.0",  
       "minimum_index_compatibility_version" : "6.0.0-beta1"
     },
     "tagline" : "You Know, for Search"
   }
   ```
    即为成功！
    再尝试通过浏览器访问http://${ip}:9200/有相同的响应成功结果。  若无法访问，则检查防火墙。

---
## 安装Kibana
1. 安装步骤可参考[官方网站](https://www.elastic.co/guide/en/kibana/current/targz.html)；或按以下步骤执行

2. 使用*root*用户，上传压缩包内的**kibana-7.14.0-linux-x86_64.tar.gz**并解压。
    ```
    tar -zxvf kibana-7.14.0-linux-x86_64.tar.gz
    ```
   
3. 进入目录备份配置文件**kibana.yml**，而后修改
    ```
    cd kibana-7.14.0-linux-x86_64/config/
    cp kibana.yml kibana.yml.bak
    vim kibana.yml 
    ------------------------------
    server.port: 5601
    server.host: "${该服务器的ip}"
    elasticsearch.hosts: ["http://${ES服务所在IP}:9200"]
    elasticsearch.username: "kibana_system"
    ------------------------------
    ```
   
4. 进入/bin 目录启动
    ```
    cd ../bin
    ./kibana
    ```
    界面会打印日志，最后出现如下所示内容，即为成功！
    ```
    Kibana is now available
    ```
    可通过浏览器访问 ```http://ip:5601```
---
## 配置权限(使用用户名和密码身份验证运行本地集群)
1. 可以参考官网[最低安全性设置](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/security-minimal-setup.html)

2. 停止 Kibana 和 Elasticsearch（如果它们正在运行）

3. 将 xpack.security.enabled 设置添加到**$ES_PATH_CONF/elasticsearch.yml**文件并将值设置为true
   ```
    cd /home/es/elasticsearch-7.14.0/config
    vim elasticsearch.yml
   -----添加以下信息-------
   xpack.security.enabled:true
   ----------------------
   ```
   *tips:该$ES_PATH_CONF变量是 Elasticsearch 配置文件的路径。如果您使用存档分发版（zip或tar.gz）安装了 Elasticsearch ，则该变量默认为$ES_HOME/config. 如果您使用软件包发行版（Debian 或 RPM），则该变量默认为/etc/elasticsearch.*

4. 启动ES，等待启动成功
   ```
   cd ../bin
   ./elasticsearch
   ```
5. 打开另一个终端窗口，进入ES目录，执行命令启用配置密码工具
    ``` 
    ./bin/elasticsearch-setup-passwords interactive
    ```
    执行后根据命令行提示配置密码

6. 配置 Kibana 以使用密码连接到 Elasticsearch ，创建 Kibana 密钥库并添加安全设置：
    ```
    cd kibana-7.14.0-linux-x86_64/
     ./bin/kibana-keystore create
     ./bin/kibana-keystore add elasticsearch.password
    ```
    出现提示时，输入*kibana_system*用户的密码。

7. 重启 Kibana,并在浏览器 “http://${kibanaIp}:5601” 以*elastic*用户身份登录 Kibana 。
    ```
    ./bin/kibana
    ```
--- 
## 附录
1.  解决ES启动报错问题
    1. 参考 [How to disable geoip usage in 7.14.0](https://discuss.elastic.co/t/how-to-disable-geoip-usage-in-7-14-0/281076)
    方案：使用 cluster settings API  而不是 elasticsearch.yml; 即安装好Kibana后，执行
    ```
    PUT _cluster/settings
    {
        "persistent": {
           "ingest.geoip.downloader.enabled": false
        }
    }
    ```
   