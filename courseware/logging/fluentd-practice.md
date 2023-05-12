###### 实践一：实现业务应用日志的收集及字段解析

目标：收集容器内的nginx应用的access.log日志，并解析日志字段为JSON格式，原始日志的格式为：

```bash
$ tail -f access.log
...
53.49.146.149 1561620585.973 0.005 502 [27/Jun/2019:15:29:45 +0800] 178.73.215.171 33337 GET https
```

收集并处理成：

```json
{
    "serverIp": "53.49.146.149",
    "timestamp": "1561620585.973",
    "respondTime": "0.005",
    "httpCode": "502",
    "eventTime": "27/Jun/2019:15:29:45 +0800",
    "clientIp": "178.73.215.171",
    "clientPort": "33337",
    "method": "GET",
    "protocol": "https"
}
```



思路：

- 配置fluent.conf
  - 使用@tail插件通过监听access.log文件
  - 用filter实现对nginx日志格式解析
- 启动fluentd服务
- 手动追加内容至access.log文件
- 观察本地输出内容是否符合预期
- input -> filter

`fluent.conf`

```bash
<source>
	@type tail
	@label @nginx_access
	path /var/log/nginx/access.log
	pos_file /var/log/nginx/nginx_access.posg
	tag nginx_access
	format none
	@log_level trace
</source>
<label @nginx_access>
   <filter  nginx_access>
       @type parser
	   key_name message
	   format  /(?<serverIp>[^ ]*) (?<timestamp>[^ ]*) (?<respondTime>[^ ]*) (?<httpCode>[^ ]*) \[(?<eventTime>[^\]]*)\] (?<clientIp>[^ ]*) (?<clientPort>[^ ]*) (?<method>[^ ]*) (?<protocol>[^ ]*)/
   </filter>
   <match  nginx_access>
     @type stdout
   </match>
</label>
```

启动服务，追加文件内容：

```bash
$ docker run -u root --rm -ti quay.io/fluentd_elasticsearch/fluentd:v3.1.0 sh
/ # cat /etc/fluent/fluent.conf
/ # mkdir /etc/fluent/config.d
/ # fluentd -c /etc/fluent/fluent.conf
/ # echo '53.49.146.149 1561620585.973 0.005 502 [27/Jun/2019:15:29:45 +0800] 178.73.215.171 33337 GET https' >>/var/log/nginx/access.log
```

使用该网站进行正则校验： [http://fluentular.herokuapp.com](http://fluentular.herokuapp.com/) 



###### 实践二：使用ruby实现日志字段的转换及自定义处理

```bash
<source>
	@type tail
	@label @nginx_access
	path /var/log/nginx/access.log
	pos_file /var/log/nginx/nginx_access.posg
	tag nginx_access
	format none
	@log_level trace
</source>
<label @nginx_access>
   <filter  nginx_access>
       @type parser
	   key_name message
	   format  /(?<serverIp>[^ ]*) (?<timestamp>[^ ]*) (?<respondTime>[^ ]*) (?<httpCode>[^ ]*) \[(?<eventTime>[^\]]*)\] (?<clientIp>[^ ]*) (?<clientPort>[^ ]*) (?<method>[^ ]*) (?<protocol>[^ ]*)/
   </filter>
   <filter  nginx_access>   
	   @type record_transformer
	   enable_ruby
       <record>
		host_name "#{Socket.gethostname}"
        my_key  "my_val"
        tls ${record["protocol"].index("https") ? "true" : "false"}
       </record>
   </filter>
   <match  nginx_access>
     @type stdout
   </match>
</label>
```

