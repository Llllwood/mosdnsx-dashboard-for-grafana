## 一. 所使用工具
1. **pmkol/mosdns-x**，**irinesistiana/mosdns** *(v4.5.3)*
    - `pmkol/mosdns-x`为二进制文件，mosdns核心，注意，不适配原版mosdns-x，仅适配[此仓库修改编译后的版本](https://github.com/Llllwood/mosdns-x)
    - `irinesistiana/mosdns`为docker容器镜像
    - 也可以不使用容器部署mosdns-x，直接运行二进制文件
2. **prom/prometheus** *(v3.0.0)*,为 `5.grafana` 提供数据源服务
3. **grafana/loki** *(main-92a4dda)*,为 `5.grafana` 提供数据源服务
4. **timberio/vector** *(nightly-2026-03-03-alpine)*，解析 `1.mosdns` 日志文本，推送至 `3.loki`
5. **grafana/grafana-image-tags** *(13.0.0-22707891732-amd64)*，向`2.prometheus `、`3.loki`查询数据，webUI面板展示

## 二. 信息传递主要流程
#### `grafana`所展示信息有两个来源
1.  `prometheus`的配置文件`prometheus.yml`中指向的`mosdnsAPI` ，为下列提供信息查询支持
    -   `DNS 总查询数`
    -   `查询错误总数`
    -   `当前 Cache 条目数`
    -   `DNS 平均响应延时`
    -   `QPS`
    -   `DNS P95 响应延时`
    -   `各场景查询 QPS`
    -   `缓存命中次数`
2.  `mosdns`将日志输出到文本并持久化映射后，再映射给`vector`读取该日志文本，`vector`将其解析转换为`loki`格式并推送至`loki`，为下列提供信息查询支持
    -   `总 Cache 命中率`
    -   `拦截域名TOP 20`
    -   `查询域名TOP 20`
    -   `客户端查询TOP 20`
    -   `本地查询TOP 20`
    -   `远程查询TOP 20`
    -   `命中缓存TOP 20`
## 三. 各工具配置文件准备
1. **pmkol/mosdns-x**
[mosdns-config.yaml](../config-file/mosdns/config.yaml)
    - 太长，不在这里说明了，主要说明与`easymosdns`自带配置文件的区别
        ```yaml

        # 这里外挂了其他插件
        include:
            - "./info_log.yaml"

        data_providers:
            - tag: chinalist
                file: ./rules/china_domain_list.txt
                auto_reload: true
            ...
            ...
            - tag: hosts
                file: ./hosts.txt
                auto_reload: true
            # 这里新增了一个文件加载路径
            - tag: custompollutedomain
                file: ./custom_pollute_domain.txt
                auto_reload: true

        plugins:
            - tag: cache_lan
                type: cache
                args:
                size: 8192
                #redis: "redis://127.0.0.1:6379/0"
                lazy_cache_ttl: 86400
                cache_everything: true
                lazy_cache_reply_ttl: 1
                when_hit: "log_query_cache_hit" # 这里插入新增的日志插件
                
            - tag: cache_wan
                type: cache
                args:
                size: 131072
                compress_resp: true
                #redis: "redis://127.0.0.1:6379/0"
                lazy_cache_ttl: 86400
                cache_everything: true
                lazy_cache_reply_ttl: 5
                when_hit: "log_query_cache_hit" # 这里插入新增的日志插件

            - ... # 其他
            - ... # 其他
            - tag: main_sequence
                type: sequence
                args:
                exec:
                    # ========== 新增：第一步执行metrics统计 ==========
                    - global_metrics
                    - log_query_statr   # 这里插入新增的日志插件
                    - hosts

                    - .. # 其他
                    - .. # 其他

                    # 这里插入新增的自定义 已知的污染域名用分流服务器或远程服务器解析
                    # 用不到可以删除
                    - if: query_is_custom_pollute_domain
                        exec:
                            # 优先返回ipv4结果
                            - _prefer_ipv4
                            - ecs_global
                            ...
                    ... # 其他
                    ... # 其他

                    # 屏蔽广告域名
                    - if: query_is_ad_domain
                        exec:
                            - black_hole
                            - ttl_1h
                            - log_query_is_ad   # 这里插入新增的日志插件
                            - _return

                    # 已知的本地域名或CDN域名用本地服务器解析
                    - if: "(query_is_local_domain) || (query_is_cdn_cn_domain)"
                        exec:
                            # 默认用本地服务器
                            - forward_local
                            - ttl_1m
                            # 预防已知的本地域名临时污染
                            - if: response_has_gfw_ip
                            exec:
                                - ecs_local
                                - forward_easymosdns
                            - log_query_is_local_resolve   # 这里插入新增的日志插件
                            - _return

                    # 已知的污染域名用分流服务器或远程服务器解析
                    - if: query_is_non_local_domain
                        exec:
                            # 优先返回ipv4结果
                            - _prefer_ipv4
                            - ecs_global
                            - primary:
                                # 默认用分流服务器
                                - forward_easymosdns
                            secondary:
                                # 超时用远程服务器
                                - forward_remote
                            fast_fallback: 2500
                            always_standby: false
                            - ttl_5m
                            - log_query_is_remote_resolve   # 这里插入新增的日志插件
                            - _return
                    ... # 其他
        ```
        - 关于`新增global_metrics，用于api查询`，为在`pmkol/easymosdns`提供的`config.yaml`上增加，请保留它，否则你需要修改上述查询的排序
        - 关于`自定义 已知的污染域名用分流服务器或远程服务器解析`，为在`pmkol/easymosdns`提供的`config.yaml`上增加
            -   `custom_pollute_domain.txt`文本
            -   `- tag: query_is_custom_pollute_domain`插件
            -   及其在`main_sequence`中的调用`- if: query_is_custom_pollute_domain`
            -   即使你使用不到，也应该保留它，可将`custom_pollute_domain.txt`清空，否则你需要修改上述查询的排序
        - 关于新增的各`log_query_...`，原版日志需将等级设为`debug`才能拿到各项信息，因此修改了一下源码，主要以`info`输出各`log_query_...`项日志，避免因debug造成日志文件过大
2. **prom/prometheus**

    [prometheus-prometheus.yml](../config-file/prometheus/prometheus.yml)

3. **loki**

    [loki-config.yaml](../config-file/loki/config.yaml)

4. **vector**

    [vector-config.yaml](../config-file/vector/config.yaml)

## 四. Docker容器部署
#### 推荐使用docker compose部署

[docker-compose.yaml](../docker-compsoe.yaml)

## 五. grafana添加数据源及导入面板
1.  打开`grafana`web面板`http://ip:15014`
2.  设置密码，默认账号`admin`、密码`admin`，首次登录需要修改密码
3.  点击右上角头像 `profile`，进入后在
    - `Preferences`.`Timezone`选择时区，选择`Asia/Shanghai`
    - `Preferences`.`Language`选择语言
    - 点击`Save preferences`保存
4.  点击左侧菜单，选择`链接` → `数据源`
    -   点击`添加新的数据源`，搜索`Prometheus`，选择
        -   `名称`保持默认，禁用`默认`开关
        -   `Connection` ：`http://prometheus:9090`
        -   `Authentication`全部保持默认，即空
        -   其他全部保持默认
        - 记录下浏览器地址栏端口号后的`/connections/datasources/edit/dff3szdzugd1ca`最后部分`dff3szdzugd1ca`
    -   点击`添加新的数据源`，搜索`loki`，选择
        -   `名称`保持默认，禁用`默认`开关
        -   `Connection` ：`http://loki:3100`
        -   `Authentication`全部保持默认，即空
        -   其他全部保持默认
        - 记录下浏览器地址栏端口号后的`/connections/datasources/edit/cff3tqpk14tmoc`最后部分`cff3tqpk14tmoc`
5.  修改`表板JSON文件`，逐一将`uid`的值修改为上一个步骤记录的值
    ```json
      "datasource": {
        "type": "prometheus", //以及 "type": "loki",
        "uid": "cerwl3npr2whsc"
      }
    ````
6.  点击左侧菜单，选择`仪表板` → 点击`创建数据面板` → 点击`导入仪表板` → 点击`上传仪表板JSON文件`上传 → 点击`更改uid`(随意长度一直的字符) → 点击 `import`。注意，此`uid`非`loki`数据源的`uid`

7.  [参考截图](images/index.md)