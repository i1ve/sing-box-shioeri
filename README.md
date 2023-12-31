# sing-box

The universal proxy platform.

[![Packaging status](https://repology.org/badge/vertical-allrepos/sing-box.svg)](https://repology.org/project/sing-box/versions)

## Documentation

https://sing-box.sagernet.org

## Support

https://community.sagernet.org/c/sing-box/

## License

```
Copyright (C) 2022 by nekohasekai <contact-sagernet@sekai.icu>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <http://www.gnu.org/licenses/>.

In addition, no derivative work may use the name or imply association
with this application without prior consent.
```

## Proxy Providers 支持

编译时加入 tag ```with_proxyprovider```

使用：
```
{
    "proxyproviders": [ // Proxy Providers 配置，参考下方，若只有一项，可省略[]
        {
            "tag": "proxy-provider-x", // 标签，必填，用于区别不同的 proxy-provider，不可重复，设置后outbounds会暴露一个同名的selector出站
            "url": "https://www.google.com", // 订阅链接，必填，仅支持Clash订阅链接
            "cache_file": "/tmp/proxy-provider-x.cache", // 缓存文件，选填，强烈建议填写，可以加快启动速度
            "force_update": "4h", // 强制更新间隔，选填，若当前缓存文件已经超过该时间，将会强制更新
            "request_timeout": "10s", // 订阅请求超时时间，选填，若不填写，将会使用默认超时时间 10s，格式：Golang time.Duration，如 10s, 1m, 1h, 1h30m, 1h30m30s
            "ip": "1.1.1.1", // 请求的IP，选填，若不填写，将会使用DNS字段中的DNS服务器
            "http3": true, // 是否使用HTTP/3，选填，实验性，可能会有奇怪的问题，对于节点订阅地址使用了CloudFlare CDN（或者支持HTTP/3的服务器），可以尝试开启
            "dns": "tcp://223.5.5.5", // 请求的DNS服务器，选填，若不填写，将会选择默认DNS，支持(udp/tcp/dot/doh/doh3/doq)
            "tag_format": "proxyprovider - %s", // 如果有多个订阅并且订阅间存在重名节点，可以尝试使用，其中 %s 为占位符，会被替换为原节点名。比如：原节点名："HongKong 01"，tag_format设置为 "PP - %s"，替换后新节点名会更变为 "PP - HongKong 01"，以解决节点名冲突的问题
            "filter": { // 过滤节点，选填
                "rule": [
                    "到期"
                ], // 过滤规则，选填，若只有一项，可省略[]
                "white_mode": false // 白名单模式（只保留匹配的节点），选填，若不填写，将会使用黑名单模式（只保留未匹配的节点）
            },
            "request_dialer": {}, // 请求的Dialer，选填，详见sing-box dialer字段，不支持detour, domain_strategy, fallback_delay
            "dialer": {}, // 节点的Dialer，选填，详见sing-box dialer字段
            "custom_group": [ // 自定义分组，选填，若只有一项，可省略[]，设置后outbounds会暴露一个同名的出站
                {
                    "tag": "selector-1", // outbound tag，必填
                    "type": "selector", // outbound 类型，必填，仅支持selector, urltest
                    "rule": [], // 节点过滤规则，选填，详见上filter.rule字段
                    "white_mode": false, // 节点过滤模式，选填，详见上filter.white_mode字段
                    ... // selector或urltest的其他字段，选填
                },
                ...
            ]
        },
        { // 示例，改tag, url可用
            "tag": "proxy-provider",
            "url": "https://www.google.com", // 订阅链接
            "cache_file": "/etc/proxy-provider-1.cache", // 缓存文件
            "force_update": "6h", // 强制更新间隔
            "dns": "tcp://223.5.5.5" // 可改用非tcp/udp，更加安全，但注意时间同步
            "filter": {
                "rule": ["到期", "剩余", "重置"]
            }
        }
    ],
    "outbounds": [...
    ]
}
```

更多：
```
1. 强制更新订阅到缓存文件（强烈建议设置缓存文件，这可以大幅加快启动速度，定时更新可使用系统crontab计划任务）

sing-box update-proxyprovider [-t 可指定proxy-provider tag, 支持多个tag]

2. 根据订阅生成出站(outbound)配置文件

sing-box show-proxyprovider [-t 可指定proxy-provider tag, 支持多个tag]

3. DNS格式 （{}中内容可以省略）
- udp://223.5.5.5{:53} （使用223.5.5.5，53端口 UDP DNS）
- tcp://223.5.5.5{:53} （使用223.5.5.5，53端口 TCP DNS）
- tls://8.8.8.8{:853} （使用8.8.8.8，853端口 TLS DNS）
- https://1.1.1.1{:443}/dns-query （使用1.1.1.1，443端口 HTTPS DNS）
- h3://1.1.1.1{:443}/dns-query （使用1.1.1.1，443端口 HTTPS DNS，使用（HTTP/3））
- quic://94.140.14.140{:784} （使用94.140.14.140，784端口 QUIC DNS）

* 不允许使用基于域名的DNS。使用基于域名的DNS服务器，依然需要使用基于IP的DNS服务器作为解析域名的DNS服务器

4. 若订阅地址响应慢，可以自行设置 request_timeout 字段，指定请求超时时间，设置为空则默认 10s 请求超时，格式：Golang time.Duration，如 10s, 1m, 1h, 1h30m, 1h30m30s

5. 若订阅地址响应慢，不建议设置 force_update，否则会导致订阅更新失败而无法启动。可以在定时机制中（如 Linux Crontab、Windows 计划任务）设置定期执行 sing-box update-proxyprovider [-c 配置文件] [-t 指定 proxyprovider tag] 强制更新（无视 force_update 规则），即使更新失败，sing-box 依照沿用旧的订阅信息启动
```
