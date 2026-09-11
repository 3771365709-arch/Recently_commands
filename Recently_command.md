
# 1. 清理嵌套仓库
git rm -f --cached DFT_TEST_TOOLS_web

# 2. 清理已暂存的缓存
git rm -r --cached __pycache__

# 3. 首次提交
git add .
git commit -m "初始化 LB3 测试数据工具仓库"

# 4. 配置远程
git remote add origin https://github.com/3771365709-arch/DFT_TEST_TOOLS.git

# 5. 首次推送
git push -u origin main

# 6. 提交 .gitignore 的后续改动（本次待办）
git add .gitignore
git commit -m "移除 .gitignore 中 DFT_TEST_TOOLS_web 条目"
git push



# PowerShell 查看 / 内存 / 磁盘 / 进程 命令速查

> **⚠️ 本文所有命令均基于 Windows 系统下的 PowerShell 环境**（Windows 10/11、Windows Server 均适用）。
> 打开方式：`Win + X` → 选「Windows PowerShell / 终端」，或 `Win + R` 输入 `powershell` 回车。
> 注意区分：部分命令（如 `systeminfo`、`tasklist`）是 CMD 也通用的外部命令；`Get-*` 是 PowerShell 原生 Cmdlet。两者在 PowerShell 里都能用。

---

## 一、内存（Memory）

### 1. 查看总内存与可用内存

```powershell
Get-CimInstance Win32_OperatingSystem |
    Select-Object CSName,
        @{n='总内存(GB)';e={[math]::Round($_.TotalVisibleMemorySize/1MB,2)}},
        @{n='可用内存(GB)';e={[math]::Round($_.FreePhysicalMemory/1MB,2)}},
        @{n='已用内存(GB)';e={[math]::Round(($_.TotalVisibleMemorySize-$_.FreePhysicalMemory)/1MB,2)}}
```

### 2. 查看内存使用率（百分比）

```powershell
$os = Get-CimInstance Win32_OperatingSystem
"内存使用率: {0:N1}%" -f (100 - ($os.FreePhysicalMemory / $os.TotalVisibleMemorySize * 100))
```

### 3. 查看内存条（插槽）硬件信息

```powershell
Get-CimInstance Win32_PhysicalMemory |
    Select-Object Manufacturer, PartNumber,
        @{n='容量(GB)';e={$_.Capacity/1GB}}, Speed, DeviceLocator
```

### 4. 性能计数器方式（实时值）

```powershell
Get-Counter '\Memory\Available MBytes'          # 可用内存(MB)
Get-Counter '\Memory\% Committed Bytes In Use'  # 提交内存使用率
```

> 英文/中文系统计数器路径名可能不同，报错时可改用 `Get-CimInstance` 方式。

---

## 二、CPU 与硬件概况

```powershell
# CPU 型号、核心数、逻辑处理器数
Get-CimInstance Win32_Processor |
    Select-Object Name, NumberOfCores, NumberOfLogicalProcessors, MaxClockSpeed

# 当前 CPU 负载百分比
(Get-CimInstance Win32_Processor | Measure-Object -Property LoadPercentage -Average).Average

# 一条命令看整机概况（Windows PowerShell 5.1 可用，输出较慢请耐心等待）
Get-ComputerInfo | Select-Object CsName, WindowsProductName, OsArchitecture, CsTotalPhysicalMemory
```

传统外部命令（CMD 通用，PowerShell 下也可运行）：

```cmd
systeminfo
```

---

## 三、磁盘

```powershell
# 各分区容量与剩余空间
Get-Volume | Where-Object DriveLetter |
    Select-Object DriveLetter, FileSystemLabel,
        @{n='总容量(GB)';e={[math]::Round($_.Size/1GB,1)}},
        @{n='剩余(GB)';e={[math]::Round($_.SizeRemaining/1GB,1)}}

# 物理磁盘信息
Get-Disk | Select-Object Number, FriendlyName, @{n='容量(GB)';e={$_.Size/1GB}}, PartitionStyle

# 目录占用排行（当前目录下最大的10个文件夹）
Get-ChildItem -Directory | ForEach-Object {
    [PSCustomObject]@{ Name=$_.Name; SizeMB=[math]::Round((Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum/1MB,1) }
} | Sort-Object SizeMB -Descending | Select-Object -First 10
```

---

## 四、进程（Process）

### 1. 查看全部进程

```powershell
Get-Process                 # 别名: ps
Get-Process | Format-Table -AutoSize
```

### 2. 按内存占用排序 —— 占用最高的前 10 个进程

```powershell
Get-Process | Sort-Object WS -Descending | Select-Object -First 10 |
    Select-Object Id, ProcessName,
        @{n='内存(MB)';e={[math]::Round($_.WS/1MB,1)}}, CPU
```

> PowerShell 7 可简写为 `Sort-Object WS -Descending -Top 10`，但 Windows 自带的 5.1 不支持 `-Top`，上面这种写法两边通用。

### 3. 按 CPU 时间排序

```powershell
Get-Process | Where-Object CPU -gt 0 | Sort-Object CPU -Descending |
    Select-Object -First 10 Id, ProcessName, CPU
```

### 4. 查找指定进程（支持通配符）

```powershell
Get-Process -Name chrome            # 精确名称
Get-Process -Name *python*          # 模糊匹配
Get-Process -Id 12345               # 按 PID 查
```

### 5. 结束进程

```powershell
Stop-Process -Name notepad -Force        # 按名称结束
Stop-Process -Id 12345 -Force            # 按 PID 结束
```

对应的外部命令（CMD 通用）：

```cmd
tasklist                      # 列出进程（/svc 可看服务关联）
taskkill /PID 12345 /F        # 按 PID 强制结束
taskkill /IM notepad.exe /F   # 按名称强制结束
```

### 6. 查看进程的命令行、路径等详细信息（含其他用户进程需管理员权限）

```powershell
Get-CimInstance Win32_Process |
    Select-Object ProcessId, Name, ExecutablePath, CommandLine
```

---

## 五、服务（可选补充）

```powershell
Get-Service                                 # 全部服务
Get-Service -Name wuauserv                  # 指定服务（此处为 Windows 更新）
Get-Service | Where-Object Status -eq 'Running'   # 只看正在运行的
Restart-Service -Name wuauserv              # 重启服务（需管理员）
```

---

## 六、网络相关（可选补充）

```powershell
ipconfig /all                               # 网络配置（CMD 通用）
Get-NetIPAddress -AddressFamily IPv4        # 本机 IPv4 地址
netstat -ano                                # 端口占用与对应 PID（CMD 通用）
ping 8.8.8.8                                # 连通性测试
Test-Connection -ComputerName www.github.com  # PowerShell 原生 ping
```

---

## 七、常用技巧与注意事项

1. **管理员权限**：右键 PowerShell 选「以管理员身份运行」。查看系统级进程、结束系统服务、安装软件等操作需要它。
2. **执行策略报错**：第一次运行 `.ps1` 脚本若提示禁止运行，管理员执行：
   ```powershell
   Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
   ```
3. **管道 `|`**：把上一个命令的输出传给下一个命令，如 `Get-Process | Sort-Object WS -Descending`，这是 PowerShell 最常用的组合方式。
4. **别名**：`ps` = `Get-Process`，`gps` 同义；`kill` = `Stop-Process`。用 `Get-Alias` 查看全部别名。
5. **看帮助**：`Get-Help Get-Process -Examples` 可查看任意命令的示例用法。
6. **wmic 已弃用**：网上老教程的 `wmic cpu get name` 等命令在较新 Windows 上已移除，请统一改用本文的 `Get-CimInstance`。
7. **PowerShell 版本**：用 `$PSVersionTable` 查看。Windows 自带的 5.1 已可用；`Get-ComputerInfo`、`Sort-Object -Top` 等在 PowerShell 7 中体验更好。


# 护眼模式使用说明（Ubuntu GNOME / Wayland）

> 本机环境：Ubuntu + GNOME 桌面 + Wayland 会话
> 控制方式：`gsettings` 命令行（等效于「设置 → 显示 → 夜灯」）

## 一、常用命令

### 1. 开启护眼模式（夜灯）

```bash
gsettings set org.gnome.settings-daemon.plugins.color night-light-enabled true
```

### 2. 关闭护眼模式

```bash
gsettings set org.gnome.settings-daemon.plugins.color night-light-enabled false
```

### 3. 调节色温（可选）

色温数值越小越黄（护眼效果越强），默认 4000，建议 3500 左右。

```bash
# 设置为 3500
gsettings set org.gnome.settings-daemon.plugins.color night-light-temperature 3500
```

## 二、查看当前状态

```bash
# 查看是否开启（true=开启，false=关闭）
gsettings get org.gnome.settings-daemon.plugins.color night-light-enabled

# 查看当前色温
gsettings get org.gnome.settings-daemon.plugins.color night-light-temperature
```

## 三、图形界面方式

「设置 → 显示 → 夜灯」中可开启/关闭，并拖动滑块调节色温。

## 四、注意事项

- 夜灯在 GNOME 中默认按日出日落自动调度，若只需手动开启，可在设置中关闭「计划」选项。
- 若环境不是 GNOME 桌面，可改用第三方工具：`redshift`（临时生效：`redshift -O 4500`）。






# Linux 磁盘/文件查看、容量管理与进程管理命令大全

> 适用范围：日常服务器运维、故障排查、磁盘空间清理、进程管理。
> 示例均以 root 或普通用户可直接执行为准，需要 root 的命令已标注 `sudo`。

---

## 目录

1. [磁盘与分区信息查看](#一磁盘与分区信息查看)
2. [磁盘容量查看（df / du）](#二磁盘容量查看df--du)
3. [文件与目录详细信息查看](#三文件与目录详细信息查看)
4. [磁盘 I/O 与性能监控](#四磁盘-io-与性能监控)
5. [内存与交换分区查看](#五内存与交换分区查看)
6. [进程查看](#六进程查看)
7. [进程管理（启动/终止/优先级）](#七进程管理启动终止优先级)
8. [后台任务、定时任务与开关机](#八后台任务定时任务与开关机)
9. [常用组合技巧与排查思路](#九常用组合技巧与排查思路)

---

## 一、磁盘与分区信息查看

### 1.1 lsblk —— 查看块设备（磁盘/分区）树状结构

```bash
lsblk                  # 树状列出所有块设备
lsblk -f               # 额外显示文件系统类型、UUID、挂载点
lsblk -d -o NAME,SIZE,ROTA,MODEL   # 只列磁盘本身，SIZE 大小，ROTA 是否机械盘，MODEL 型号
```

### 1.2 fdisk / parted —— 查看分区表

```bash
sudo fdisk -l                  # 列出所有磁盘及分区的详细信息
sudo fdisk -l /dev/sda         # 只看指定磁盘
sudo fdisk /dev/sda            # 进入交互式分区管理（m 查看帮助，注意勿误操作）
sudo parted -l                 # 支持 GPT 分区表的查看，2TB 以上大硬盘推荐
sudo parted /dev/sda print     # 查看指定磁盘分区表
```

### 1.3 blkid / findmnt / mount —— 查看文件系统与挂载

```bash
sudo blkid                     # 查看分区 UUID、文件系统类型
findmnt                        # 树状显示当前挂载点
mount | column -t              # 查看已挂载的文件系统（对齐排版）
cat /proc/mounts               # 内核视角的挂载信息
cat /etc/fstab                 # 查看开机自动挂载配置
sudo mount /dev/sdb1 /mnt/data # 挂载分区
sudo umount /mnt/data          # 卸载挂载点
sudo mount -a                  # 重新挂载 fstab 中所有条目（用于验证配置）
```

### 1.4 硬盘健康状态（SMART）

```bash
sudo smartctl -a /dev/sda      # 查看硬盘 SMART 全部信息（需安装 smartmontools）
sudo smartctl -H /dev/sda      # 只看健康状态快速结论
```

### 1.5 LVM 相关

```bash
sudo pvs / vgs / lvs           # 查看物理卷/卷组/逻辑卷概要
sudo pvdisplay / vgdisplay / lvdisplay   # 查看详细信息
sudo df -h /dev/mapper/centos-root       # 查看 LVM 逻辑卷使用情况
```

### 1.6 创建文件系统与检查修复（mkfs / fsck / fstrim）

```bash
sudo mkfs.ext4 /dev/sdb1          # 格式化为 ext4
sudo mkfs.xfs /dev/sdc1           # 格式化为 xfs（大文件、高并发场景常用）
sudo mkfs.vfat -F 32 /dev/sdd1    # 格式化为 FAT32（U 盘等）

sudo umount /dev/sdb1             # fsck 必须对未挂载的分区执行
sudo fsck -y /dev/sdb1            # 自动检查并修复文件系统（root 分区需进救援模式操作）

sudo fstrim -av                   # SSD 手动 TRIM，回收已删除数据块，维持长期写入性能
```

### 1.7 数据迁移与备份（rsync / dd）

```bash
# rsync：增量同步，比 cp 可靠（只传差异、可续传、可限速）
rsync -av --progress /data/ /backup/data/               # 本地同步（末尾斜杠很关键：带 / 是同步“目录内容”，不带 / 会把 data 目录本身拷过去）
rsync -avz --delete /data/ root@192.168.1.10:/backup/   # 推到远程主机（走 SSH），--delete 使两边完全一致
rsync -av --bwlimit=50M /data/ /backup/                 # 限速 50MB/s，避免占满磁盘 I/O
rsync -av --exclude='*.log' /data/ /backup/             # 排除指定文件
rsync -av --partial --append /data/ /backup/            # 大文件断点续传

# dd：整盘/分区克隆与镜像备份（⚠️ of 指向谁，谁的数据就被覆盖，执行前务必 lsblk 确认设备名！）
sudo dd if=/dev/sda of=/dev/sdb bs=4M status=progress              # 整盘克隆到另一块盘
sudo dd if=/dev/sda of=/backup/sda.img bs=4M status=progress       # 整盘备份为镜像文件
dd if=/dev/zero of=/data/test.img bs=1M count=1024 oflag=direct    # 测真实写入速度（oflag=direct 绕过缓存），测完删除测试文件
sudo dd if=/data/test.img of=/dev/null bs=1M iflag=direct          # 测读取速度
```

### 1.8 磁盘配额（多用户限制空间）

```bash
sudo repquota -a                                             # ext4：查看所有用户配额使用情况（挂载时需带 usrquota 参数）
sudo xfs_quota -x -c 'report -h' /data                       # XFS：查看 /data 配额报告（挂载时需 uquota/gquota 参数）
sudo xfs_quota -x -c 'limit bsoft=5g bhard=10g yang' /data   # 设置用户 yang 软上限 5G、硬上限 10G
```

### 1.9 软 RAID 状态查看

```bash
cat /proc/mdstat                  # 内核视角的 RAID 状态与重建/同步进度
sudo mdadm --detail /dev/md0      # 详细状态：UU 表示正常，U_ 表示有一块盘掉线
```

### 1.10 硬件信息查看

```bash
lscpu                             # CPU 架构、核心数、主频、缓存
sudo dmidecode -t memory          # 内存条插槽、单条容量、频率
sudo dmidecode -t system          # 整机厂商、序列号（虚拟机可借此识别宿主平台）
sudo smartctl -i /dev/sda         # 硬盘型号、序列号、固件版本
```

---

## 二、磁盘容量查看（df / du）

### 2.1 df —— 查看文件系统整体磁盘占用

```bash
df             # 默认以 KB 为单位显示
df -h          # 以人类可读单位（K/M/G/T）显示，最常用
df -HT         # 同时显示文件系统类型
df -i          # 查看 inode 使用情况（小文件过多导致“磁盘没满却写不进”时用）
df -h /home    # 只看指定目录所在分区
df -x tmpfs    # 排除 tmpfs 等虚拟文件系统
df -a          # 显示所有文件系统（含 0 块的）
```

输出字段说明：

| 字段 | 含义 |
|------|------|
| Filesystem | 文件系统（设备名） |
| Size | 总容量 |
| Used | 已用容量 |
| Avail | 可用容量 |
| Use% | 使用率 |
| Mounted on | 挂载点 |

### 2.2 du —— 查看目录/文件占用的空间大小

```bash
du                     # 默认列出当前目录下每个子目录的大小（KB）
du -h                  # 人类可读单位
du -sh /var            # 只显示 /var 的总大小（-s 汇总，最常用）
du -h --max-depth=1 /  # 只统计第一层子目录大小，逐层排查大目录
du -sh *               # 当前目录下各项的大小并排序前先看一眼
du -ah /etc            # 列出目录中所有文件及子目录的大小
du -sh --exclude="*.log" /var  # 统计时排除某些文件
du -h -t 100M /        # 只显示大于 100M 的目录/文件
```

### 2.3 快速定位大文件 / 大目录（空间排查三板斧）

```bash
# 1) 从根开始找最大的 10 个目录
sudo du -h / --max-depth=1 2>/dev/null | sort -rh | head -n 10

# 2) 全盘查找大于 500MB 的文件并按大小排序
sudo find / -type f -size +500M -exec ls -lh {} \; 2>/dev/null | awk '{print $5, $9}' | sort -rh

# 3) 找出 /var/log 下最大的文件（日志清理常用）
sudo find /var/log -type f -size +100M -exec ls -lh {} \;

# 交互式分析工具（需安装 ncdu，强烈推荐）
sudo ncdu /
```

### 2.4 磁盘清理常用命令

```bash
sudo du -sh /var/log/* | sort -rh | head       # 找出最大的日志
sudo journalctl --disk-usage                    # 查看 journal 日志占用
sudo journalctl --vacuum-size=200M              # 将 journal 日志压缩到 200M
sudo truncate -s 0 /var/log/huge.log            # 清空大日志文件（不要直接 rm 正在写的日志）
sudo rm -rf /var/cache/yum/*                    # 清理 yum 缓存（apt 对应 /var/cache/apt/archives/*）
sudo apt clean / sudo yum clean all             # 清理包管理器缓存
```

> ⚠️ 注意：对仍被进程占用的日志文件，用 `truncate -s 0` 清空，不要 `rm`，否则空间不会释放（进程仍持有文件句柄），可用 `lsof | grep deleted` 查看被删除但仍占空间的文件。

---

## 三、文件与目录详细信息查看

### 3.1 ls —— 查看文件列表与属性

```bash
ls -l          # 长格式：权限、硬链接数、属主、属组、大小、修改时间、文件名
ls -lh         # 长格式 + 人类可读大小
ls -la         # 包含隐藏文件（. 开头）
ls -lt         # 按修改时间倒序
ls -ltr        # 按修改时间正序（最新在最后，看日志目录很方便）
ls -lS         # 按文件大小排序
ls -lh --time-style=long-iso   # 时间显示为 2026-09-09 10:00 格式
ls -i          # 显示 inode 号
ls -R          # 递归列出子目录
```

`ls -l` 输出示例解读：

```
-rw-r--r--  1 root root  4096 Sep  9 10:23 file.txt
│   │  │ │     │    │     │        │          └─ 文件名
│   │  │ │     │    │     │        └─ 修改时间
│   │  │ │     │    │     └─ 大小(字节)
│   │  │ │     │    └─ 属组
│   │  │ │     └─ 属主
│   │  │ └─ 其他人权限
│   │  └─ 属组权限
│   └─ 属主权限
└─ 文件类型（-普通文件 d目录 l链接 c字符设备 b块设备 s套接字 p管道）
```

### 3.2 stat / file —— 查看文件元数据

```bash
stat file.txt          # inode、权限、时间戳（atime/mtime/ctime）、大小等全部元数据
stat -c '%s %n' *.log  # 自定义格式输出大小和文件名
file report.pdf        # 判断文件类型（文本/二进制/压缩包/图片等）
file -i data.txt       # 显示 MIME 类型和编码
```

### 3.3 查看文件内容

```bash
cat /etc/hosts             # 输出全部内容
cat -n file.txt            # 带行号显示
tac file.txt               # 倒序输出（cat 反过来）
head -n 20 file.log        # 看前 20 行
tail -n 50 file.log        # 看后 50 行
tail -f /var/log/messages  # 实时跟踪日志新增内容（最常用）
tail -F app.log            # 同 -f，但日志被轮转(重建)后仍继续跟踪
less +F app.log            # less 模式下实时跟踪，按 Ctrl+C 可回翻浏览
more file.txt              # 分页查看
wc -l file.txt             # 统计行数；-w 单词数；-c 字节数
```

### 3.4 查找文件

```bash
find /etc -name "*.conf"              # 按名称找
find / -iname "Readme"                # 忽略大小写
find /var -type f -size +100M         # 找大于 100M 的文件
find . -mtime -1                      # 24 小时内修改过的文件（+1 是 1 天前）
find /tmp -type f -empty              # 找空文件
find . -type d -name logs             # 找目录
find / -user nginx -type f            # 按属主找
which python3                         # 查找可执行命令路径
whereis nginx                         # 查找命令、源码、man 手册位置
locate nginx.conf                     # 基于数据库快速查找（updatedb 更新库）
```

### 3.5 lsof —— 查看被打开的文件（文件与进程的桥梁）

```bash
lsof /var/log/messages        # 查看哪些进程打开了该文件
lsof -p 1234                  # 查看进程 1234 打开的所有文件
lsof -u nginx                 # 查看 nginx 用户打开的文件
lsof -i :80                   # 查看占用 80 端口的进程
lsof +D /var/log              # 递归查看目录下被打开的文件（大目录较慢）
lsof | grep deleted           # 找出“已删除但仍被进程占用”的文件（空间不释放元凶）
```

### 3.6 文件隐藏属性（lsattr / chattr）

```bash
lsattr file.txt               # 查看文件的隐藏属性
sudo chattr +i file.txt       # 加 i 锁定：不可修改、删除、改名（root 也不行）
sudo chattr -i file.txt       # 去掉 i 锁定后才能删除
sudo chattr +a app.log        # a 属性：只允许追加内容，日志防篡改常用
```

> 排查场景：`rm` 报 `Operation not permitted` 且属主权限都正常时，先 `lsattr` 看是否被加了 `i` 锁。

### 3.7 文件完整性校验（md5sum / sha256sum）

```bash
md5sum file.iso               # 生成 MD5 校验值
sha256sum file.iso            # 生成 SHA256（更安全，推荐）
md5sum -c file.iso.md5        # 按清单文件校验
find /data -type f -exec sha256sum {} \; > checksums.txt    # 批量生成校验清单
sha256sum -c checksums.txt | grep -v ': OK'                 # 批量校验，只显示异常项
```

---

## 四、磁盘 I/O 与性能监控

### 4.1 watch —— 周期性刷新任意命令

```bash
watch -n 2 df -h                                # 每 2 秒刷新一次 df
watch -n 1 -d 'ss -s'                           # -d 高亮每次变化的部分
watch -n 5 'ls -lh /mnt/data'                   # 盯拷贝进度、空间变化
watch -n 2 'ps aux --sort=-%cpu | head -n 8'    # 盯 CPU 占用变化
```

### 4.2 磁盘 I/O 实时监控

```bash
iostat -x 1 5         # 每 1 秒输出一次共 5 次，-x 显示扩展统计（%util、await 等）
iostat -d -k 2        # 只看设备级 I/O，KB 单位，每 2 秒刷新
vmstat 1              # 综合监控：CPU、内存、swap、io（bi/bo 列）
sar -d 1 5            # 采集磁盘活动（需 sysstat）
sar -u 1 5            # CPU 使用率历史/实时
sar -r 1 5            # 内存使用率
sar -n DEV 1 5        # 网卡流量
iotop                 # 按进程查看实时磁盘读写（类似 top 的 I/O 版，需 sudo）
pidstat -d 1          # 按进程统计 I/O，每秒刷新
dstat -cdngy 1        # 综合监控 CPU/磁盘/网络/系统
hdparm -Tt /dev/sda   # 简单测试磁盘读取速度
```

关键字段解读（`iostat -x`）：

| 指标 | 含义 | 经验判断 |
|------|------|----------|
| %util | 设备繁忙时间占比 | 长期接近 100% 说明磁盘是瓶颈 |
| await | I/O 平均等待时间(ms) | 机械盘一般 <20ms，持续偏高异常 |
| r/s, w/s | 每秒读/写次数 | 结合业务判断是否异常 |

---

## 五、内存与交换分区查看

```bash
free -h               # 人类可读查看内存与 swap（最常用）
free -m               # MB 单位
free -h -s 2          # 每 2 秒刷新
swapon --show         # 查看启用的交换分区
cat /proc/meminfo     # 内核详细内存信息
cat /proc/swaps       # swap 明细
```

### 5.1 创建 swap 文件完整流程（小内存云主机常用）

```bash
sudo fallocate -l 4G /swapfile        # 创建 4G swap 文件（fallocate 不可用时改用 dd if=/dev/zero of=/swapfile bs=1M count=4096）
sudo chmod 600 /swapfile              # 权限必须是 600
sudo mkswap /swapfile                 # 格式化为 swap
sudo swapon /swapfile                 # 启用
swapon --show && free -h              # 验证生效
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # 写入 fstab，开机自动启用
sudo swapoff /swapfile                # 不需要时关闭
```

`free -h` 输出解读：

| 字段 | 含义 |
|------|------|
| total | 物理内存总量 |
| used | 已用（total - free - buff/cache） |
| free | 完全未使用 |
| buff/cache | 系统缓存（可回收，不算“真占用”） |
| available | 真正可供新程序使用的内存，**看这个才准** |

---

## 六、进程查看

### 6.1 ps —— 静态查看进程快照

```bash
ps aux                # BSD 风格：列出所有进程（USER/%CPU/%MEM/STAT/COMMAND）
ps -ef                # System V 风格：全格式列表（UID/PID/PPID/STIME/CMD）
ps -ef --forest       # 树状显示进程父子关系
ps aux --sort=-%cpu   # 按 CPU 占用降序（找 CPU 杀手）
ps aux --sort=-%mem   # 按内存占用降序（找内存杀手）
ps -u nginx           # 只看某用户的进程
ps -p 1234 -o pid,ppid,user,%cpu,%mem,cmd   # 指定 PID，自定义输出列
ps -eLf               # 查看线程（L 显示 LWP 线程号）
ps -eo pid,cmd --no-headers | wc -l   # 统计进程总数
```

`ps aux` 的 STAT 状态码：

| 状态 | 含义 |
|------|------|
| R | 正在运行 |
| S | 可中断睡眠（等待事件） |
| D | 不可中断睡眠（通常是 I/O 等待，杀不掉） |
| Z | 僵尸进程（父进程未回收） |
| T | 已停止（被暂停/被跟踪） |
| `<` | 高优先级；`N` 低优先级 |
| `s` | 会话组长；`l` 多线程；`+` 前台进程 |

### 6.2 top —— 实时动态监控（交互式）

```bash
top               # 默认 3 秒刷新
top -p 1234       # 只监控指定 PID
top -u nginx      # 只看某用户进程
top -d 1          # 1 秒刷新一次
top -b -n 1 > top.txt   # 批处理模式输出一次快照（写脚本时用）
```

top 内部常用交互键：

| 按键 | 作用 |
|------|------|
| `P` | 按 CPU 排序 |
| `M` | 按内存排序 |
| `T` | 按累计运行时间排序 |
| `k` | 输入 PID 杀进程 |
| `r` | 修改进程 nice 值 |
| `1` | 展开显示每颗 CPU 核心 |
| `H` | 显示线程 |
| `u` | 按用户过滤 |
| `f` | 自定义显示列 |
| `W` | 保存当前配置 |
| `q` | 退出 |

### 6.3 htop / atop —— 增强版监控

```bash
htop    # 彩色、支持鼠标、方向键选择进程、F9 杀进程（需安装）
atop    # 记录历史资源快照，可回溯排查（atop -r /var/log/atop/atop_20260909）
glances # 一屏总览 CPU/内存/磁盘/网络/进程（需安装）
```

### 6.4 pstree / pgrep / pidof —— 快速定位进程

```bash
pstree                 # 树状显示全部进程
pstree -p              # 附带 PID
pstree -p 1234         # 显示某进程的子进程树
pgrep nginx            # 按名称输出 PID 列表
pgrep -l nginx         # 同时显示进程名
pgrep -u yang -l bash  # 按用户+名称查找
pidof sshd             # 精确按程序名取 PID
pgrep -f "python app.py"   # -f 匹配完整命令行
```

### 6.5 查看进程详情与系统负载

```bash
uptime                 # 开机时长 + 1/5/15 分钟负载
cat /proc/1234/status  # 进程 1234 的详细状态（内存、线程数等）
cat /proc/1234/cmdline # 进程完整启动命令
ls -l /proc/1234/cwd   # 进程的工作目录
ls -l /proc/1234/exe   # 进程对应的可执行文件
pmap -x 1234           # 进程内存映射详情
strace -p 1234         # 跟踪进程的系统调用（排查卡顿时用）
ltrace -p 1234         # 跟踪库函数调用
systemd-cgtop          # 按 cgroup（服务）查看资源占用
```

### 6.6 排查进程“莫名消失”（OOM killer）

```bash
sudo dmesg -T | grep -iE 'oom|out of memory|killed process'   # 查 OOM 记录（-T 显示可读时间）
journalctl -k | grep -i oom      # systemd 系统的等价方式
sudo dmesg -T | tail -n 50       # 最近的内核消息
```

> 内存耗尽时内核 OOM killer 会直接杀掉占内存最大的进程，进程自己的日志里往往查不到原因，只有内核日志留有记录。处理方向：给业务加内存限制、加 swap，或用 `echo -500 | sudo tee /proc/<PID>/oom_score_adj` 降低该进程被杀的优先级。

---

## 七、进程管理（启动/终止/优先级）

### 7.1 kill —— 按 PID 发送信号

```bash
kill 1234              # 默认发 SIGTERM(15)，温和终止，允许进程清理后退出
kill -9 1234           # SIGKILL，强制杀死（最后手段，不给进程清理机会）
kill -15 1234          # 同默认，温和终止
kill -1 1234           # SIGHUP，常用于让服务重新加载配置
kill -2 1234           # SIGINT，等同前台 Ctrl+C
kill -l                # 列出所有信号
kill -0 1234           # 不发信号，仅检测进程是否存在/有无权限（脚本常用）
```

常用信号速查：

| 信号 | 编号 | 作用 |
|------|------|------|
| SIGHUP | 1 | 挂起/重载配置 |
| SIGINT | 2 | 键盘中断 Ctrl+C |
| SIGQUIT | 3 | 退出并转储核心 |
| SIGKILL | 9 | 强制终止，不可被捕获 |
| SIGTERM | 15 | 请求终止，可被捕获清理（默认） |
| SIGSTOP | 19 | 暂停进程 |
| SIGCONT | 18 | 恢复暂停的进程 |

### 7.2 pkill / killall —— 按名称批量结束进程

```bash
pkill nginx            # 按名称杀掉所有 nginx 进程
pkill -9 -f "python app.py"    # 强杀命令行匹配的进程
pkill -u yang          # 杀掉某用户的所有进程
killall nginx          # 按精确程序名杀进程（无匹配会报错，比 pkill 严格）
killall -i nginx       # 交互式确认后逐个杀
```

> ⚠️ `pkill` 默认是模糊匹配进程名，误杀风险较高，建议先 `pgrep -l` 确认再杀；`-f` 会匹配完整命令行，更要谨慎。

### 7.3 服务管理与服务日志（systemctl / journalctl）

```bash
sudo systemctl stop nginx      # 由 systemd 负责优雅停止（生产服务推荐方式）
sudo systemctl restart nginx   # 重启
sudo systemctl reload nginx    # 仅重载配置
sudo systemctl status nginx    # 查看服务状态
sudo systemctl list-units --failed   # 列出启动失败的服务
```

```bash
journalctl -u nginx                        # 查看某服务的全部日志
journalctl -u nginx -f                     # 实时跟踪服务日志（等同 tail -f）
journalctl -u nginx -n 100 --no-pager      # 最近 100 行，不进翻页器
journalctl -u nginx --since "1 hour ago"   # 只看最近 1 小时（也支持 "2026-09-09 10:00:00"）
journalctl -p err -b                       # 本次开机以来 error 及以上级别的日志
```

### 7.4 调整进程优先级

```bash
nice -n 10 ./backup.sh         # 以 nice=10（低优先级）启动，减少抢 CPU
sudo nice -n -10 ffmpeg ...    # 提高优先级需要 root
renice -n 5 -p 1234            # 修改运行中进程的优先级
sudo renice -n -5 -p 1234      # 降低 nice 值（提高优先级）需 root
ionice -c 3 -p 1234            # 限制进程 I/O 调度优先级（空闲时才读写）
```

> nice 范围 -20（最高优先级）~ 19（最低优先级），数值越大越“谦让”。

### 7.5 僵尸进程处理

```bash
ps aux | awk '$8=="Z"'         # 找出僵尸进程
# 僵尸进程本身杀不掉，需让其父进程回收；父进程不配合时只能杀父进程：
kill -15 <父进程PID>
```

### 7.6 进程资源限制（ulimit / prlimit）

```bash
ulimit -a                     # 查看当前会话全部资源限制
ulimit -n                     # 最大打开文件数（too many open files 报错先看这里）
ulimit -n 65535               # 临时调高，仅对当前 shell 及其子进程生效
cat /proc/1234/limits         # 查看运行中进程的实际限制
prlimit --pid 1234 --nofile=65535:65535   # 直接修改运行中进程的限制
lsof -p 1234 | wc -l          # 与 ulimit -n 对比，确认是否逼近上限
```

持久化配置三处（按生效范围选）：

```bash
# 1) 用户级：/etc/security/limits.conf
#    *  soft  nofile  65535
#    *  hard  nofile  65535
# 2) systemd 服务：/etc/systemd/system/nginx.service.d/limits.conf
#    [Service]
#    LimitNOFILE=65535      （改后执行 systemctl daemon-reload 并重启服务）
# 3) 全局默认：/etc/systemd/system.conf 的 DefaultLimitNOFILE
```

---

## 八、后台任务、定时任务与开关机

### 8.1 后台作业与 nohup

```bash
./long_task.sh &        # 放到后台运行
Ctrl + Z                # 将前台任务暂停并放到后台（stopped 状态）
jobs                    # 查看当前 shell 的后台作业
jobs -l                 # 附带 PID
bg %1                   # 让 1 号后台作业继续在后台运行
fg %1                   # 把 1 号作业调回前台
nohup ./server.sh &     # 挂断终端（退出登录）后进程不退出，输出到 nohup.out
nohup ./server.sh > app.log 2>&1 &   # 重定向标准输出与错误到指定日志（标准写法）
disown -h %1            # 已启动但没用 nohup 的任务，用 disown 免疫挂断
setsid ./server.sh &    # 在新会话中运行，脱离当前终端
screen / tmux           # 终端复用器：断线重连后任务继续（长任务推荐）
```

### 8.2 timeout —— 给命令限时

```bash
timeout 60 ./task.sh             # 60 秒未结束则发 SIGTERM 终止
timeout -k 10 60 ./task.sh       # TERM 后 10 秒仍不退出则强杀（SIGKILL）
timeout --signal=9 30 ./task.sh  # 直接指定 SIGKILL；超时终止的退出码为 124
```

### 8.3 定时任务（crontab）

```bash
crontab -l                       # 查看当前用户的定时任务
crontab -e                       # 编辑当前用户的定时任务
sudo crontab -l -u nginx         # 查看指定用户的任务
systemctl status crond           # 定时服务状态（Debian/Ubuntu 服务名是 cron）
```

```bash
# crontab 格式：分 时 日 月 周 命令，示例：
# 30 2 * * *   /usr/bin/find /var/log -type f -mtime +30 -delete         # 每天 2:30 清理 30 天前的日志
# */10 * * * * /usr/local/bin/check_disk.sh > /var/log/check.log 2>&1   # 每 10 分钟执行一次并记录输出
```

### 8.4 安全关机与重启（sync / shutdown / reboot）

```bash
sync                             # 将内存缓存数据强制刷盘，拔盘/关机前先执行
shutdown -h now                  # 立即关机
shutdown -h +10 "10分钟后维护关机"   # 定时关机并向所有登录用户广播提示
shutdown -r now                  # 立即重启（等价 reboot）
shutdown -c                      # 取消已计划的关机
```

---

## 九、常用组合技巧与排查思路

### 9.1 一键定位“磁盘为什么满了”

```bash
df -h                                          # ① 找到满的分区
du -h -x --max-depth=1 / 2>/dev/null | sort -rh | head -n 10   # ② 逐层下钻
sudo find / -xdev -type f -size +500M 2>/dev/null | xargs ls -lhS   # ③ 找大文件
sudo lsof | grep deleted | sort -k7 -rh | head # ④ 检查已删除但未释放空间的文件
df -i                                          # ⑤ inode 是否耗尽（海量小文件）
```

### 9.2 一键定位“CPU/内存被谁吃了”

```bash
ps aux --sort=-%cpu | head -n 6        # CPU Top5
ps aux --sort=-%mem | head -n 6        # 内存 Top5
top -b -n 1 | head -n 20               # top 快照
ps -eo pid,ppid,%cpu,%mem,cmd --sort=-%cpu | head -n 6   # 带父子关系
cat /proc/meminfo | grep -E "MemTotal|MemAvailable|SwapTotal|SwapFree"
```

### 9.3 实用别名建议（写入 ~/.bashrc）

```bash
alias df='df -h'
alias du='du -h'
alias free='free -h'
alias ll='ls -alFh'
alias port='ss -tulnp'
alias meminfo='free -h; echo; ps aux --sort=-%mem | head -n 6'
alias cpumem='ps aux --sort=-%cpu | head -n 6'
```

### 9.4 端口与进程联动排查

```bash
ss -tulnp | grep 8080          # 查看谁在监听 8080
sudo lsof -i :8080             # 同上（lsof 方式）
netstat -tulnp | grep 3306     # 老系统用 netstat
fuser -v 8080/tcp              # 查看 8080 端口进程
```

### 9.5 进阶工具一览（了解即可，按需安装）

| 工具 | 用途 |
|------|------|
| `strace -c -T -p PID` | 统计进程系统调用的次数与耗时，定位进程卡在哪一步 |
| `perf top` | 实时分析 CPU 热点函数（性能调优） |
| `cat /proc/pressure` | PSI：内核级 CPU/内存/IO 压力指标（4.20+ 内核） |
| `taskset -pc 0-3 PID` | 把进程绑定到指定 CPU 核心 |
| `inotifywait -m /path` | 实时监控目录的文件变化事件 |
| `systemd-analyze blame` | 各服务开机耗时排序，排查开机慢 |
| `duf` / `dust` / `btop` | df / du / top 的现代彩色替代品 |

---

## 附：命令速查总表

| 场景 | 命令 |
|------|------|
| 磁盘整体使用率 | `df -h` |
| inode 使用情况 | `df -i` |
| 目录占用大小 | `du -sh` / `du -h --max-depth=1` |
| 找大文件 | `find / -type f -size +500M` |
| 磁盘/分区信息 | `lsblk` / `fdisk -l` / `parted -l` |
| 磁盘 I/O 监控 | `iostat -x 1` / `iotop` |
| 文件详细信息 | `ls -lh` / `stat` / `file` |
| 实时跟踪日志 | `tail -f` / `less +F` |
| 打开的文件/端口 | `lsof` / `ss -tulnp` |
| 内存 | `free -h` |
| 进程快照 | `ps aux` / `ps -ef` |
| 实时进程监控 | `top` / `htop` |
| 找进程 PID | `pgrep -l` / `pidof` |
| 杀进程 | `kill` / `pkill` / `killall` |
| 服务管理 | `systemctl status/stop/start/restart` |
| 调优先级 | `nice` / `renice` |
| 后台常驻 | `nohup cmd > log 2>&1 &` / `tmux` |
| 服务日志跟踪 | `journalctl -u 服务名 -f` |
| 进程被 OOM 杀掉 | `dmesg -T \| grep -i oom` |
| 周期刷新监控 | `watch -n 2 'df -h'` |
| 格式化 / 修复 | `mkfs.ext4` / `fsck -y` |
| 数据迁移备份 | `rsync -av --progress` / `dd` |
| 资源上限 | `ulimit -a` / `prlimit --pid` |
| 文件锁定 / 隐藏属性 | `lsattr` / `chattr +i` |
| 文件完整性校验 | `sha256sum` |
| 命令限时 | `timeout 60 cmd` |
| 定时任务 | `crontab -e` |
| 关机重启 | `shutdown -h now` / `shutdown -r now` |
