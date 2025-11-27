# Dify_configuration
Dify 本地化部署的一些配置项参考

# 1.配置sandbox中的python第三方库

- 导出pip的requirements文件，并将其中的内容复制进 **docker/volumes/sandbox/dependencies/python-requirements.txt** 文件中，如下:

```text
bcrypt==4.3.0
certifi==2025.1.31
cffi==1.17.1
charset-normalizer==3.4.1
cryptography==44.0.2
idna==3.10
ldap3==2.9.1
msal==1.32.0
ntlm-auth==1.5.0
paramiko==3.5.1
pyasn1==0.6.1
pycparser==2.22
pycryptodome==3.22.0
PyJWT==2.10.1
PyNaCl==1.5.0
requests==2.32.3
scapy==2.6.1
urllib3==2.4.0
beautifulsoup4==4.13.3
```

- 容器启动时会自动下载这些包到sandbox中
  
## 2. 增加ssrf_proxy的目的端口访问范围

- 移除 **docker/ssrf_proxy/squid.conf.template** 中以下行的注释，让ssrf_proxy可以访问除了443以外的https端口
```text
  acl SSL_ports port 1025-65535   # Enable the configuration to resolve this issue: https://github.com/langgenius/dify/issues/12792
```

### 3. 将dify的 *HTTP 请求* 模块中的ssl verify去除，保证可以访问所有站点

- 将 **docker/.env**文件中的以下参数修改为：
```text
    HTTP_REQUEST_NODE_SSL_VERIFY=False #使得*HTTP 请求模块不验证目标SSL证书
```

#### 4. 修改sandbox的 syscall 解除系统调用的限制

- 将 **docker/volumes/sandbox/conf/config.yaml** 中的 *allowed_syscalls* 修改为
  `allowed_syscalls: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, ...]

##### 5. 修改sandbox 允许访问的外部网络

- sandbox的http请求通过SSRF_PROXY访问到外部站点。但是SSRF_PROXY是个http代理，不能代理其他协议，同时sanbox默认禁止直接访问外部网络。因此需要配置sandbox[...]
  
  - 编辑 **docker/docker-compose.yaml** 配置文件找到**sandbox**的配置，在**networks**处，添加default,如下：
  - *原配置*：
  
    ```yaml

    networks:
      - ssrf_proxy_network

    ```
  - *修改后配置*:
    
    ```yaml

    networks:
      - ssrf_proxy_network
      - default

    ```

###### 6. 修改 PluginDaemon 的超时时间
- 工作流中如果某节点是使用插件执行长阻塞操作，会报错 *PluginDaemonInternalServerError: killed by timeout* ，这是因为PluginDaemon运行插件默认超时时间过短，需[...]
- 
  ```text
  PLUGIN_MAX_EXECUTION_TIMEOUT=86400
  ```

###### 7. 修改代码执行模块最大输出长度
- 代码执行模块默认输出字符数量有限制，超过数量会报错，需要修改 **docker/.env** 配置增加输出长度，修改如下参数：
      ```text
      CODE_MAX_STRING_LENGTH=16000000
      CODE_MAX_STRING_ARRAY_LENGTH=300000
      CODE_MAX_OBJECT_ARRAY_LENGTH=300000
      CODE_MAX_NUMBER_ARRAY_LENGTH=300000
      ```

###### 8. 修改工作流最长执行时间
- 工作流默认只能执行1200秒，到时会抛异常*stopped by user*，需要修改 **/docker/.env** 配置，修改如下参数:

      `WORKFLOW_MAX_EXECUTION_TIME=86400`
      `APP_MAX_EXECUTION_TIME=86400`

###### 9. 修改Sandbox worker及代码执行模块最大执行时间
- 由于代码执行模块依赖sandbox，而sandbox中单个worker的执行时间默认最大为15秒，超时后就会被强制kill，所以需要增加Sandbox worker最大执行时间。同时，��[...]
      ```text
      SANDBOX_WORKER_TIMEOUT=7200
      CODE_EXECUTION_READ_TIMEOUT=7200
      ```

###### 10. 修改工作流最大执行步数
- 工作流默认最大步数500步，到时会抛异常*Max steps 500 reached.*，需要修改**docker/.env** 配置，修改如下参数:

      `WORKFLOW_MAX_EXECUTION_STEPS=50000`

###### 11. 修改工作流嵌套深度
- 工作流中可以嵌入其他工作流，默认只能嵌套5层，否则报异常*Max workflow call depth 5 reached.*，可以调整**docker/.env**增加嵌套层数：

      `WORKFLOW_CALL_MAX_DEPTH=10`
###### 12. 修改PIP源
- dify安装的插件很多都依赖python包，默认的pip源是官方源，速度慢，需要修改，可以在**docker/.env**中修改以下参数：
      `PIP_MIRROR_URL=https://pypi.tuna.tsinghua.edu.cn/simple`

###### 13. 修改迭代模块在并行模式下最大的参数提交数量
- 迭代模块在并行模式下，输入参数提交到线程池的数量有默认100的限制（数量计算公式为："输入数组长度×设置的并行数>100"），触发这个后会报错 "Ma[...]
      `MAX_SUBMIT_COUNT=50000` #注意，修改此属性会导致服务器资源压力增大，实际情况下要根据服务器配置调整此参数
###### 14. 修改API 服务 Server worker 数量，即 gevent worker 数量，公式：cpu 核心数 x 2 + 1
- 提高性能，调整**docker/.env**以下字段：
      `SERVER_WORKER_AMOUNT=9` #4 CPU Core
###### 15. 调整日志级别和日志时区
- 日志级别有`DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`，默认时区为UTC，修改**docker/.env** 以下字段
      `LOG_LEVEL=ERROR`
      `LOG_TZ=Asia/Shanghai`
###### 16. 设置请求处理超时时间
- 请求处理超时时间，默认 360，酌情增加，以支持更长的 sse 连接时间，修改**docker/.env** 以下字段
      `GUNICORN_TIMEOUT=900`
###### 17. 设置HTTP读写的最大超时时间
- 防止MCP长时间执行导致timeout，修改**docker/.env** 以下字段
      `HTTP_REQUEST_MAX_READ_TIMEOUT=900`
      `HTTP_REQUEST_MAX_WRITE_TIMEOUT=900`

###### 17. 启动dify

- 在 **docker/** 目录下运行:
    `docker compose up -d` 即可启动dify
