# n8n VPS：配置怎么选、一年要花多少钱，Vultr 小时计费自建 Docker 自动化的避坑指南

搜 n8n VPS 的人，想解决的大多是同一件事：让 n8n 在一台自己说了算的服务器上 7×24 跑着，执行多少次都不另外收费。先把结论放在最前面——n8n 对硬件要求很低，1–2 GB 内存的机型就能扛住个人自动化，选服务器时真正该比的是计费方式、长期价格和机房位置，而不是参数表上最抢眼的那个数字。

n8n 是节点式工作流自动化工具，自带几百个现成集成节点，源码可查、支持完全自托管。工具本身不收执行费，你付的只有 VPS 一项。所以这篇文章围绕三个实际问题展开：配置买多大、一年真金白银花多少、部署要折腾多久。

> **选购速览**：个人自动化场景，Vultr 的 2 GB 机型（High Performance 系列，$12/月）是最省心的选择，执行次数不限、随时销毁；想先试水，开一台 1 GB 的 $6 机型，按小时计费，跑一两天成本不到 1 美元。部署走 Docker Compose + PostgreSQL，顺利的话半小时跑通。👉 [查看 Vultr 全套餐当前价格](https://www.vultr.com/pricing/?ref=7268283)

## n8n 需要多大的 VPS？先算清自己的执行量

个人用 1–2 GB 内存就够，小团队建议 4 GB 起步。判断依据是执行量：一条每分钟触发一次的工作流，一个月就是 60×24×30 = 43,200 次执行。放在 Zapier 这类按任务计费的平台上，这个量级一个月要几百美元；放在自建 VPS 上，它只是占用一点 CPU 时间。

对照下面的场景挑配置，基本不会出错：

- 每天几十次以内的低频自动化：1 GB 机型（$5–6/月）够用，建议配一点 swap
- 个人或两三人团队，每天几百次执行：2 GB（$12/月），n8n 和 PostgreSQL 同机跑正合适
- 工作流多、webhook 收发频繁、带数据处理：4 GB（$24/月）起步
- 每天数千次执行：8 GB 以上（$48/月），把 n8n 切到队列模式，主进程、Redis 和多个 worker 分担负载

内存是第一优先级。n8n 本体是 Node.js 进程，生产环境还要再搭一个 PostgreSQL，两者挤在 1 GB 里比较紧，2 GB 是舒服的下限。CPU 反而在其次——n8n 平时基本空闲，webhook 来了瞬间处理完就交还资源。

## Vultr 的套餐和价格：三个系列，小时计费

Vultr 的计费方式和多数 VPS 卖家不同：按小时计费，费用封顶不超过对应月价。开一台 2 GB 机型跑一天大约 0.4 美元，测试完销毁就停止计费，没有包年合同。买错配置的代价被压到最低，这是它对 n8n 自建新手最友好的一点。

Cloud Compute 分三个系列，价差主要在 CPU 类型和存储介质：

| 系列 | 配置 | 月流量 | 月付（美元） | 开通 |
| --- | --- | --- | --- | --- |
| Regular Performance（Intel） | 1 vCPU / 1 GB / 25 GB SSD | 1 TB | $5.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| Regular Performance | 1 vCPU / 2 GB / 55 GB SSD | 2 TB | $10.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| Regular Performance | 2 vCPU / 4 GB / 80 GB SSD | 3 TB | $20.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| Regular Performance | 4 vCPU / 8 GB / 160 GB SSD | 4 TB | $40.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Performance（AMD EPYC） | 1 vCPU / 1 GB / 25 GB SSD | 1 TB | $6.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Performance | 1 vCPU / 2 GB / 60 GB SSD | 2 TB | $12.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Performance | 2 vCPU / 4 GB / 100 GB SSD | 3 TB | $24.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Performance | 4 vCPU / 8 GB / 200 GB SSD | 4 TB | $48.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Frequency（Intel + NVMe） | 1 vCPU / 1 GB / 25 GB NVMe | 1 TB | $6.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Frequency | 1 vCPU / 2 GB / 38 GB NVMe | 2 TB | $12.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Frequency | 2 vCPU / 4 GB / 64 GB NVMe | 3 TB | $24.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |
| High Frequency | 4 vCPU / 8 GB / 128 GB NVMe | 4 TB | $48.00 | [开通](https://www.vultr.com/pricing/?ref=7268283) |

更高规格还有 16 GB 到 64 GB 的档位，以及独占 vCPU 的 Optimized 系列和裸金属服务器，都在同一个价格页里能选到。

三个系列怎么挑？Regular 最便宜，CPU 是共享的 Intel 核；High Performance 换成 AMD EPYC，同价位存储给得更多；High Frequency 用 NVMe 盘，适合读写密集的场景。对 n8n 来说，三者的日常感知差距不大，**High Performance 的 2 GB / $12 档位最均衡**，60 GB 存储对一个执行数据库来说非常宽裕。👉 [查看 2 GB 机型的当前价格](https://www.vultr.com/pricing/?ref=7268283)

几项和长期使用直接相关的政策：

- 快照免费。销毁前打一个快照，数据和配置随时可以恢复
- 自动备份加收实例价格的 20%，按需开启
- 云防火墙免费，控制台里直接配规则
- 机房覆盖 30 多个位置，亚洲有东京、首尔、新加坡可选
- 付款支持信用卡、PayPal，也有支付宝

## 对比其他 n8n VPS 卖家：便宜的和省心的各是什么价

搜 n8n VPS 时，常见的卖家不止 Vultr 一家。把几家放在一张表里对比（各家价格随时会调，下单前以官网标价为准）：

| 卖家 | 入门价格 | 主打特点 |
| --- | --- | --- |
| Vultr | 1 GB $6/月；2 GB $12/月 | 按小时计费随时销毁，30 多个机房，快照免费 |
| Cloudzy | 2 GB 标价 $7.48/月（当前五折促销，原价 $14.95/月） | 14 天退款，12 个机房，99.95% 在线率 SLA |
| LightNode | 2 GB $7.71/月 | 40 多个机房，同样支持按小时计费，99.95% 在线率保证 |
| BuyVPS | 2 GB $7.99/月（首月 $1） | 预配置 n8n + PostgreSQL + Caddy，机房仅阿姆斯特丹、纽约两处，流量不限 |
| IONOS | 4 GB 前 3 个月 $5/月，之后 $14/月（一年期合约） | n8n 预装开机即用，欧洲机房 |

看完这张表，有三个判断值得单独说。

先分清标价和长期价。IONOS 的 $5 只给前 3 个月，之后回到 $14/月；Cloudzy 页面上的数字是当前五折活动的结果。Vultr 标价 $12 就是长期价，第 12 个月付的和第 1 个月一样。比较两家总价时，别拿促销价乘以 12。

流量这件事对 n8n 几乎无所谓。webhook 报文一般只有几 KB，就算一个月跑几十万次执行，流量也就几个 GB。所以“不限流量”这类卖点不值得为它多付钱，真正该比的是内存、存储和机房到你常用 API 的延迟。

各家真正拉开差距的是省心程度。完全不想碰命令行的话，IONOS 和 BuyVPS 的预装方案开机就有现成的 n8n，代价是机型选择少、价格整体略高；担心买错想无理由退款，Cloudzy 有 14 天退款期。Vultr 给出的对应解法是小时计费，用一两天就知道合不合适，停掉即止，不用走退款流程。如果后续还想在同一台机器上折腾别的东西（自建面板、跑脚本、搭个小站），Vultr 这类标准云主机的可玩性也更大。

## 在 Vultr 上把 n8n 跑起来：从开机到 HTTPS

标准路径是 Docker Compose + PostgreSQL + 反向代理，顺利的话 30 分钟左右完成。按顺序来：

1. **开实例**：选 High Performance 2 GB，系统装 Ubuntu 24.04。Vultr 的 Marketplace 里有预装 Docker 的应用镜像，能省掉手动装 Docker 的一步；不想手写配置文件，也可以选 Coolify 这类自托管面板，装好后从网页上一键部署 n8n。区域怎么选：面向国内用户触发为主，东京、首尔、新加坡延迟更低；主要对接欧美 SaaS 的，选阿姆斯特丹、伦敦或纽约。👉 [查看 Marketplace 里的应用镜像](https://www.vultr.com/marketplace/?ref=7268283)
2. **系统加固**：防火墙只放行 22、80、443 三个端口，建一个普通用户来操作，别用 root 直接跑服务
3. **写 compose 文件**：两个服务，PostgreSQL 16 和官方 n8n 镜像（docker.n8n.io/n8nio/n8n）。n8n 端口只绑 127.0.0.1:5678，不要直接暴露公网
4. **设环境变量**：N8N_HOST 和 WEBHOOK_URL 填你的域名，GENERIC_TIMEZONE 设好时区，再用 `openssl rand -hex 32` 生成一个 N8N_ENCRYPTION_KEY
5. **配反代和证书**：Caddy 最省事，自动申请续期 Let's Encrypt 证书；Nginx + certbot 组合效果相同
6. **解析域名**：加一条 A 记录指向实例 IP，浏览器打开 https://你的域名，创建管理员账号就能进编辑器了

## 自建 n8n 最容易翻车的三个地方

绝大多数 n8n 自建问题都出在下面三件事上，部署前先记下来：

1. **加密密钥丢了，等于凭据全没。** N8N_ENCRYPTION_KEY 负责加密数据库里的所有凭据，它不随备份走，服务器一旦重装就找不回来。生成的当天就把它抄到服务器之外（存进密码管理器即可）——这是自建 n8n 出事最多的一步。
2. **WEBHOOK_URL 不设置，外部服务调不回来。** 少了这个变量，n8n 生成的回调地址里会带 localhost，Stripe、Slack 这类服务拿到的是打不通的 URL。典型症状是编辑器里测试一切正常，真实触发永远超时。
3. **SQLite 只用来试水。** 默认安装用的是 SQLite 文件库，数据量一大容易锁库，断电还有损坏风险。生产环境第一天就切到 PostgreSQL，再配一个每天 pg_dump 的定时任务；Vultr 的快照免费，每周打一个，等于多一层保险。

## 常见问题

**1 GB 内存跑得动 n8n 吗？**
低频自动化可以，但要加 swap，也别把 PostgreSQL 的缓存设太大。要长期稳定跑，从 2 GB 起步更省心。

**Vultr 用不满一个月怎么计费？随时能停吗？**
按小时计费、月度封顶，控制台里销毁实例即停止计费，没有合约。销毁前打一个免费快照，数据和配置都能留到下次恢复。

**自建 n8n 一年总共花多少钱？**
以 2 GB / $12 机型算：服务器 $144/年，开自动备份再加 20% 约 $29，域名通常 $10–15/年，合计 180 美元上下。对照托管平台的算法——Zapier 入门付费档大约 $30/月，只给 750 次任务额度，一年 $360；跑量越大差距越大，粗略的盈亏平衡点在每月一两千次任务附近。👉 [用小时计费先开一台测一测](https://www.vultr.com/pricing/?ref=7268283)

**除了 VPS 还要准备什么？**
一个域名（A 记录指向服务器 IP）、一个 SSH 客户端、大约半小时。没有其他前置成本。

## 最后怎么选

每天几十次以内的低频自动化，或者还在观望：开 1 GB 的 $6 机型试水，跑两天销毁，成本不到 1 美元。个人或两三人团队：2 GB High Performance（$12/月）足够，先不必考虑队列模式。执行量每天数千次：直接上 4 GB（$24/月）或 8 GB（$48/月），再拆 worker。完全不想写命令行：IONOS、BuyVPS 这类预装 n8n 的卖家开机即用，代价是选择面窄、价格略高。

下单前只有一件事要记得做：实例开机后，先别急着导入工作流，把 N8N_ENCRYPTION_KEY 存到服务器之外。这一步做完，剩下的部署基本不会再出岔子。👉 [从 Vultr 的 2 GB 机型开始](https://www.vultr.com/pricing/?ref=7268283)
