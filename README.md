# rclone vultr：Object Storage 配置、自动备份与迁移全流程，避开限流和 sync 误删这两个坑

搜 rclone vultr 的人，十有八九在做两件事之一：给 Vultr Object Storage 配 rclone 做备份，或者把数据从 Amazon S3、Google Cloud Storage 搬到 Vultr。两条路的核心动作一样——把 Vultr 当成一块 S3 兼容存储，在 rclone.conf 里写一段十几行的配置，之后全部用标准 rclone 命令操作。流程本身不难，真正容易翻车的是两个细节：sync 命令会删掉目标端“多余”的文件，以及大流量传输时可能撞上 Vultr 的接口限流。这篇文章把配置、测试、定时备份、迁移和这两个坑一次讲清。

> **选购速览**：rclone 本身免费开源，你需要的是一个 Vultr Object Storage 订阅（订阅制计费，当前档位和价格在产品页直接可见）。S3 凭证在订阅的 Overview 标签页复制，配置五分钟能跑通。👉 [查看 Vultr Object Storage 的档位与当前价格](https://www.vultr.com/products/object-storage/?ref=7268283)

## 开始之前：要准备什么

整个流程只需要三样东西：装好的 rclone、一个 Vultr Object Storage 订阅，以及订阅后台的 S3 凭证。

rclone 按平台安装：Windows 可用 Chocolatey，macOS 和 Linux 可用 Homebrew，Linux 服务器上更省事的办法是跑一遍官方安装脚本：


curl -O https://rclone.org/install.sh
sudo bash install.sh
rclone version


`rclone version` 能正常输出版本号就说明装好了。

然后是 Vultr 这边。开通一个 Object Storage 订阅，在管理页复制三样东西：主机名（endpoint，形如 ewr1.vultrobjects.com）、Access Key、Secret Key。测试阶段再建一个 bucket，比如 testbucket，就齐了。

| 你需要准备 | 说明 | 获取方式 |
| --- | --- | --- |
| Vultr Object Storage 订阅 | rclone 的远程端，S3 兼容，订阅制计费 | [查看档位与价格](https://www.vultr.com/products/object-storage/?ref=7268283) |
| S3 凭证三件套 | 主机名、Access Key、Secret Key | 订阅管理页 Overview 标签页复制 |
| rclone | 免费开源，本地或服务器都能装 | 官方安装脚本 / Homebrew / Chocolatey |
| Vultr 云服务器（可选） | 需要 7×24 挂定时备份任务时开一台 | [查看 Ubuntu 实例方案](https://www.vultr.com/servers/ubuntu/?ref=7268283) |

## rclone.conf 怎么写

配置的本质是在 rclone.conf 里给 Vultr 添加一个远程端。先用 `rclone config file` 找到配置文件路径，macOS 和 Linux 上一般是 `~/.config/rclone/rclone.conf`，文件不存在也没关系，按提示的路径新建一个就行。

写入以下内容，密钥换成你自己的：


[vultr-ewr1]
type = s3
provider = Vultr
env_auth = false
access_key_id = 你的AccessKey
secret_access_key = 你的SecretKey
region =
endpoint = https://ewr1.vultrobjects.com
location_constraint =
acl = private
server_side_encryption =
storage_class =


几个容易让人犹豫的字段，一次说清：

| 字段 | 作用 |
| --- | --- |
| type = s3 | 协议类型，Vultr Object Storage 走 S3 兼容协议 |
| provider | 供应商标识，见下方说明 |
| env_auth | 是否从环境变量读凭证；密钥已写进配置就填 false |
| access_key_id / secret_access_key | S3 密钥，订阅管理页复制 |
| region / location_constraint | Vultr 示例里留空，地区由 endpoint 表达 |
| endpoint | 订阅所在地区的主机名，如 ewr1.vultrobjects.com |
| acl | 默认文件权限，见下方说明 |
| server_side_encryption / storage_class | 留空即按默认值运行 |

关于 provider，有个值得知道的细节：Vultr 自己的教程里出现过两种写法——早期教程用 `provider = Vultr`，新教程用 `provider = Other`，两套都出自官方文档。照抄哪一套都可以，保持自己配置文件里写法统一即可，真正决定能不能连上的是 endpoint 和密钥。

acl 这行单独说。`private` 意味着只有你（所有者）拥有完整权限，其他人完全无法访问；`public-read` 则是所有人可读。Vultr 默认把 Object Storage 实例设为 PRIVATE 权限，除非你打算通过公开 URL 分享文件，否则保持 private 更稳妥。

## 先跑通：连通测试和第一个文件

配置写完，先确认 rclone 认识这个远程端：


rclone lsd vultr-ewr1:


能列出你名下的 bucket，凭证和 endpoint 就都没问题。

找个空目录传个测试文件：


mkdir tmp && cd tmp
echo "hello rclone" > test1.txt
rclone copy test1.txt vultr-ewr1:testbucket
rclone ls vultr-ewr1:testbucket


输出里的 `13 test1.txt` 表示文件已进桶，13 是字节数。这里顺手验证了一个重要行为：把本地的 test1.txt 删掉再 rclone ls，桶里那份还在。copy 只管往里加文件，不动已有的内容。

## 备份用 copy 还是 sync

一句话规则：**备份用 copy，镜像和迁移用 sync**。

- `rclone copy`：把源端文件复制到目标端，已经复制过的自动跳过，所以可以反复执行，天然适合增量备份。本地删了文件，桶里那份不受影响。
- `rclone sync`：让目标端与源端完全一致。源端删掉的文件，目标端也会跟着删。

第二条就是最常见的翻车点。sync 的语义是镜像，它对目标端“多余文件”的处理方式是直接删除——Vultr 官方文档里专门提醒过，为避免误删备份应优先使用 copy。真要用 sync，先确认源端是你唯一可信的数据源。

计划按 3-2-1 原则做备份的话（重要数据存三份、两种介质、一份异地），Object Storage 承担“异地那份”很合适，日常增量交给 copy。到这一步整个链路已经通了，剩下的就是把订阅开起来：👉 [开通 Vultr Object Storage，跑通你的第一个备份桶](https://www.vultr.com/products/object-storage/?ref=7268283)

## 定时自动备份：交给 cron

手动执行命令算不上备份，定时任务才是正经形态。rclone 装好后写进 crontab 即可：


crontab -e


加一行，比如每天凌晨 3 点把项目目录增量备份到桶里：


0 3 * * * rclone copy /home/youruser/project vultr-ewr1:backup >> /home/youruser/rclone.log 2>&1


copy 是增量模式，天天跑也不会重复传输没变化的文件，适合长期挂着。跑任务的机器需要保持开机——本来就有常开服务器的话直接加这行 crontab；没有的话，开一台便宜的实例专门干这件事也花不了多少：👉 [看看 Vultr 的 Ubuntu 云服务器方案](https://www.vultr.com/servers/ubuntu/?ref=7268283)。

## 从 Amazon S3 或 Google Cloud Storage 迁移

迁移需要配两个远程端：一个指向源（AWS 或 GCS），一个指向 Vultr。以 S3 为例，在 rclone.conf 里加上：


[aws]
type = s3
provider = AWS
access_key_id = 你的AWS密钥
secret_access_key = 你的AWS密钥
env_auth = false


[vultr-ewr1] 段保持不变，然后执行：


rclone sync aws: vultr-ewr1: --progress


这条命令会把 AWS 账户下的 bucket 迁到 Vultr，`--progress` 在终端显示实时进度。只想搬某个桶，就写成 `rclone sync aws:桶名 vultr-ewr1:桶名`。

迁移场景用 sync 而不是 copy，原因正是上一节说过的语义：迁移要的就是源和目标完全一致，前提依然是目标端没有需要保留的独立数据。

GCS 建议走交互式向导：运行 `rclone config`，按 n 新建远程端，起个名字（比如 gcs），在供应商列表里选 Google Cloud Storage，OAuth client ID 和 secret 都直接回车留空，再填入 GCP 项目编号。配好后同样用 `rclone sync gcs:桶名 vultr-ewr1:桶名` 搬数据。

数据量大时，把下一节的限流参数带上再跑。

## 撞上 429、503、504 怎么办

这三个 HTTP 错误码说明你触发了 Vultr Object Storage 的速率限制，并发越高越容易撞上。解决方法是给 rclone 限速：


rclone sync aws: vultr-ewr1: --progress --tpslimit 100 --transfers 1


`--tpslimit 100` 把每秒事务数压到 100，`--transfers 1` 关掉并行传输、改成逐个文件传。速度会慢一些，但能稳定跑完——对一次性的迁移任务来说，慢一点总比反复报错强。

## 常见问题

**rclone 连 Vultr Object Storage，必须用 Vultr 的服务器吗？**

不用。rclone 跑在你本地或任何服务器上都行，只要能安装、能联网。macOS、Linux、Windows 全支持，命令通用。用 Vultr 云服务器的好处是定时备份可以不间断地跑。

**region 和 location_constraint 要填吗？**

Vultr 官方示例里都是留空，地区信息由 endpoint 表达。各位置对应的主机名（比如 ewr1 新泽西、ams1 阿姆斯特丹）在 Vultr 的 Object Storage 文档里能查到完整列表。

**bucket 想公开访问怎么设？**

把配置里的 acl 改成 `public-read`，所有者保持完整控制权，其他人获得只读权限。只建议在确实要通过 URL 分享文件时使用，日常保持 private。

**sync 删掉的文件还能救回来吗？**

默认情况下救不回来，rclone 不会为目标端的删除保留副本。所以备份任务坚持用 copy，把 sync 留给你确认过语义的迁移和镜像场景。

**不同地区的桶之间能互相同步吗？**

能。给每个地区各写一个远程端，密钥各用各的，然后 `rclone sync ewr1:桶名 ams1:桶名`。不同远程端还可以设置不同的 acl，一个私有一个公开都没问题。

## 最后给个动手路线

按目的选路径就够了：

- **定时备份服务器数据**：一个 Object Storage 订阅 + copy + cron，acl 保持 private，全程不用碰 sync。
- **从 S3 或 GCS 搬家**：源和 Vultr 各配一个远程端，sync 全量迁移，命令带上 --tpslimit 100 和 --transfers 1 防限流。
- **只是想先试试**：开订阅、建个 testbucket，把测试那节的三条命令跑一遍，十分钟内能看到结果。

下单前确认两件事：实例在哪个地区（决定 endpoint 主机名），以及凭证从管理页 Overview 标签页复制。主机名、Access Key、Secret Key 三样齐了，剩下的就是照着配置段填空。👉 [去 Vultr 开通 Object Storage 订阅](https://www.vultr.com/products/object-storage/?ref=7268283)
