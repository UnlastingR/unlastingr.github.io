# NAT 机服务部署技术文档

> 环境：Debian 11 (bullseye) x86_64，NAT 机，无公网 80/443 端口
> 外部 IP：160.202.238.39
> 域名：*.sy.freeaswind.xyz（Cloudflare 托管，通配符 A 记录 *.sy → 160.202.238.39）

---

## 一、Docker 安装与镜像加速

### 安装 Docker
使用 LinuxMirrors 脚本，选择阿里云源：
```bash
bash <(curl -sSL https://linuxmirrors.cn/docker.sh)
```

### 配置镜像加速
```bash
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live",
    "https://docker.xuanyuan.me",
    "https://docker.unsee.tech",
    "https://docker.chenby.cn"
  ]
}
EOF
systemctl daemon-reload && systemctl restart docker
```

### ghcr.io 镜像加速（南京大学）
```bash
# 拉取时替换前缀
docker pull ghcr.nju.edu.cn/yangchuansheng/derper:v1.94.2
docker tag ghcr.nju.edu.cn/yangchuansheng/derper:v1.94.2 ghcr.io/yangchuansheng/derper:v1.94.2

```

---

## 二、代理配置

### Shell 临时代理函数
```bash
proxy() {
    local URL="socks5h://<你自己的代理>"
    if [ "$1" = "set" ]; then
        export http_proxy="$URL" https_proxy="$URL" all_proxy="$URL"
        echo "代理已开启: $URL"
    elif [ "$1" = "unset" ]; then
        unset http_proxy https_proxy all_proxy
        echo "代理已关闭"
    else
        echo "用法: proxy [set|unset]"
    fi
}
```

### Docker daemon 代理说明
- 此代理为 socks5h 协议，Docker daemon 不完整支持
- 代理不支持 CONNECT 隧道，无法代理 Docker Hub TLS 流量
- **最终方案：使用国内镜像站代替，无需配置 Docker daemon 代理**

---

## 三、端口规划

| 服务 | 内部端口 | NAT 外部端口 | 协议 |
|------|---------|------------|------|
| Lucky Web UI | 16601 | — | TCP |
| Lucky 反代监听 | 16666 | 57066 | TCP |
| dpanel | 8807 | — | TCP |
| derper DERP | 12345 | — | TCP（经 Lucky 反代） |
| derper STUN | 3478 | 3478 | TCP+UDP |
| Tailscale | 41641 | 41641 | UDP |

---

## 四、Lucky 部署

```bash
docker run -d \
  --name lucky \
  --restart always \
  --net=host \
  -v /root/lucky:/goodluck \
  gdy666/lucky:latest
```

**注意事项：**
- 使用 `--net=host` 模式
- 首次启动后需从内网访问后台开启外网访问开关
- 默认账号：`admin` / `admin666`
- 后台地址：`http://127.0.0.1:16601`

---

## 五、DNS 配置（Cloudflare）

| 类型 | 名称 | 值 |
|------|------|-----|
| A | `*.sy` | `160.202.238.39` |

通配符覆盖所有 `xxx.sy.freeaswind.xyz` 子域名，无需为每个服务单独添加记录。

---

## 六、Lucky 反向代理配置

监听端口：`16666`（NAT 外部映射到 57066）

| 域名 | 后端目标 | 备注 |
|------|---------|------|
| `lucky.sy.freeaswind.xyz` | `http://127.0.0.1:16601` | Lucky 后台 |
| `dpanel.sy.freeaswind.xyz` | `http://127.0.0.1:8807` | dpanel 面板 |
| `derp.sy.freeaswind.xyz` | `https://127.0.0.1:12345` | DERP 服务（需关闭后端证书验证） |

**关键：** derp 后端必须用 `https://`，并勾选**跳过后端 TLS 证书验证**（自签名证书）。

---

## 七、dpanel 部署

```bash
docker run -d \
  --name dpanel \
  --restart always \
  -p 127.0.0.1:8807:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /root/dpanel:/dpanel \
  donknap/dpanel:latest
```

- 默认账号：`admin` / `admin`
- 访问地址：`https://dpanel.sy.freeaswind.xyz:57066`

---

## 八、Tailscale 安装（二进制安装）

> 原因：apt 源地址含 IPv6，NAT 机无 IPv6，连接超时导致安装失败

```bash
# 用代理下载
proxy set
curl --proxy socks5h://<你自己的代理> \
  -L https://pkgs.tailscale.com/stable/tailscale_latest_amd64.tgz \
  -o tailscale.tgz && tar xzf tailscale.tgz
cd tailscale_1.94.2_amd64/

# 安装二进制
cp tailscale tailscaled /usr/local/bin/
ln -s /usr/local/bin/tailscaled /usr/sbin/tailscaled
ln -s /usr/local/bin/tailscale /usr/sbin/tailscale

# 配置 systemd 服务
cp systemd/tailscaled.service /etc/systemd/system/
cat > /etc/default/tailscaled <<EOF
PORT=41641
FLAGS=
EOF

systemctl daemon-reload
systemctl enable tailscaled
systemctl start tailscaled

# 加入网络
tailscale up --auth-key=<YOUR_AUTH_KEY> --advertise-exit-node
```

### 开启 IP 转发（exit node 必须）
```bash
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### NAT 机直连优化
在 NAT 控制台映射 `UDP 41641 → 内部IP:41641`，其他节点可与此机直连，无需经过 DERP 中继。

---

## 九、DERP 服务器部署

### 架构
```
客户端 HTTPS
    ↓ NAT 57066
Lucky :16666  ← TLS 终止，证书由 Lucky 管理自动续签
    ↓ HTTPS（自签名）
derper :12345 ← DERP 协议处理
    ↓
tailscaled.sock ← verify-clients 验证客户端身份
```

### compose.yml
```yaml
# /root/derper/compose.yml
services:
  derper:
    image: ghcr.io/yangchuansheng/derper:v1.94.2
    container_name: derper
    restart: always
    ports:
      - "127.0.0.1:12345:12345"
      - "3478:3478/udp"
      - "3478:3478/tcp"
    volumes:
      - /root/derper/cert:/cert
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock:ro
    environment:
      DERP_DOMAIN: derp.sy.freeaswind.xyz
      DERP_CERT_MODE: manual      # 无证书时自动生成 2 年期自签名证书
      DERP_CERT_DIR: /cert
      DERP_ADDR: ':12345'         # TLS 监听端口
      DERP_STUN_PORT: '3478'
      DERP_HTTP_PORT: '-1'        # 关闭 HTTP 调试端口
      DERP_VERIFY_CLIENTS: "true" # 只允许 Tailscale 网络内设备，防止蹭用
```

```bash
mkdir -p /root/derper/cert
docker compose -f /root/derper/compose.yml up -d
docker logs derper
```

**关键说明：**
- `DERP_CERT_MODE=manual`：无证书时自动生成 2 年期自签名证书
- `DERP_VERIFY_CLIENTS=true`：需挂载 `tailscaled.sock`，仅允许 Tailscale 网络内设备使用
- Lucky 反代目标填 `https://127.0.0.1:12345`，并关闭后端证书验证（自签名）
- STUN（UDP 3478）需要 NAT 控制台单独映射

---

## 十、Tailscale ACL derpMap 配置

```json
"derpMap": {
    "OmitDefaultRegions": false,
    "Regions": {
        "902": {
            "RegionID":   902,
            "RegionCode": "sy-nat",
            "RegionName": "SY NAT DERP",
            "Nodes": [
                {
                    "Name":     "derp-902",
                    "RegionID": 902,
                    "HostName": "derp.sy.freeaswind.xyz",
                    "IPv4":     "160.202.238.39",
                    "STUNPort": 3478,
                    "DERPPort": 57066
                }
            ]
        }
    }
}
```

### 验证
```bash
tailscale netcheck
# 应看到 sy-nat 出现在 DERP latency 列表，延迟极低（本机约 2.5ms）

tailscale ping <对端 Tailscale IP>
# 显示 via DERP 说明走中继，via xxx.xxx.xxx.xxx 说明直连
```

---

## 十一、常见问题排查

### Docker pull 失败（access denied）
- 原因：未配置镜像源，直连 Docker Hub 被墙
- 解决：配置 `/etc/docker/daemon.json` 镜像源列表

### Tailscale apt 安装失败
- 原因：NAT 机无 IPv6，DNS 解析到 IPv6 地址导致连接超时
- 解决：手动下载二进制包安装

### tailscaled 服务启动失败（Failed to load environment files）
- 原因：service 文件引用了 `/etc/default/tailscaled` 但文件不存在
- 解决：
```bash
cat > /etc/default/tailscaled <<EOF
PORT=41641
FLAGS=
EOF
```

### tailscaled 二进制路径不匹配
- 原因：service 文件写的是 `/usr/sbin/tailscaled`，实际装在 `/usr/local/bin/`
- 解决：
```bash
ln -s /usr/local/bin/tailscaled /usr/sbin/tailscaled
ln -s /usr/local/bin/tailscale /usr/sbin/tailscale
```

### derper 一直重启（证书找不到）
- 原因：旧镜像 `fredliang/derper` 使用 `DERP_CERT_MODE=manual` 时不会自动生成证书
- 解决：换用 `ghcr.io/yangchuansheng/derper`，会自动生成自签名证书

### Lucky 反代 derper 报 "client sent HTTP request to HTTPS server"
- 原因：Lucky 后端填的是 `http://` 但 derper 跑的是 TLS
- 解决：改为 `https://127.0.0.1:12345` 并关闭后端证书验证


---

## 十二、性能数据

| 指标                             | 数值                   |
| ------------------------------ | -------------------- |
| DERP 延迟（本机 netcheck）           | 2.5ms                |
| Windows → NAT 机 Tailscale ping | 32ms                 |
| Moonlight 串流延迟                 | 70-90ms              |
| Moonlight 码率                   | 8-15Mbps             |
| 连接方式                           | DERP 中继（NAT 穿透未成功直连） |

---

## 十三、待优化项

- [ ] Moonlight 实现 Tailscale 直连（当前走 DERP 中继）
- [ ] ethtool UDP GRO 优化（需内核 5.20+，当前 5.10 不支持）
- [ ] Lucky 证书自动导出挂载给 derper（当前用自签名）
