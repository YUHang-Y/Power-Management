
# 华为笔记本 Arch Linux 电池充电阈值自启配置
[![Arch Linux](https://img.shields.io/badge/Arch%20Linux-rolling-blue)](https://archlinux.org/)
[![Systemd](https://img.shields.io/badge/Systemd-250+-green)](https://systemd.io/)

适用于华为笔记本（MateBook 系列为主）在 Arch Linux 下，通过 systemd 实现电池充电阈值（80%）开机自动生效，解决 TLP 导致的发热问题。

## 适用场景
- 华为笔记本运行 Arch Linux（内核 ≥ 5.15，需支持电池充电阈值接口）；
- TLP 工具使用后出现发热/性能问题，需轻量化方案；
- 希望限制电池充电至 80% 以延长寿命。

## 实现思路
1. 编写轻量化脚本，直接操作内核电池控制接口；
2. 创建 systemd 服务，设置开机自启执行脚本；
3. 验证配置生效，确保阈值持久化。

## 操作步骤

### 第一步：创建充电阈值配置脚本
创建脚本文件（路径：`/usr/local/bin/set_battery_threshold.sh`）：
```bash
sudo nano /usr/local/bin/set_battery_threshold.sh
```

粘贴以下内容（已移除冗余 `sudo`，增加错误日志）：
```bash
#!/bin/bash
# 华为笔记本电池充电阈值设置脚本
# 适用：Arch Linux + 华为 MateBook 系列（内核 ≥ 5.15）
# 功能：设置电池 70% 开始充电，80% 停止充电

# 定义电池设备名（绝大多数华为本为 BAT0，可通过 ls /sys/class/power_supply/ 确认）
BATTERY="BAT0"
# 充电阈值配置
START_THRESH=70  # 低于此值开始充电
STOP_THRESH=80   # 达到此值停止充电

# 检查电池路径是否存在
if [ ! -d "/sys/class/power_supply/$BATTERY" ]; then
    echo "$(date) - 错误：未找到电池设备 $BATTERY，请检查设备名！" >> /var/log/battery_threshold.log
    exit 1
fi

# 设置充电阈值（root 权限执行，无需 sudo）
echo $START_THRESH > /sys/class/power_supply/$BATTERY/charge_control_start_threshold
echo $STOP_THRESH > /sys/class/power_supply/$BATTERY/charge_control_end_threshold

# 验证并记录日志
if [ $? -eq 0 ]; then
    START_CUR=$(cat /sys/class/power_supply/$BATTERY/charge_control_start_threshold)
    STOP_CUR=$(cat /sys/class/power_supply/$BATTERY/charge_control_end_threshold)
    echo "$(date) - 电池阈值设置成功：开始=$START_CUR%，停止=$STOP_CUR%" >> /var/log/battery_threshold.log
    echo "✅ 电池阈值设置成功：开始=$START_CUR%，停止=$STOP_CUR%"
else
    echo "$(date) - 错误：电池阈值设置失败！" >> /var/log/battery_threshold.log
    echo "❌ 电池阈值设置失败，请检查权限或内核版本！"
    exit 1
fi
```

保存并退出（`Ctrl+O` → 回车 → `Ctrl+X`），添加可执行权限：
```bash
sudo chmod +x /usr/local/bin/set_battery_threshold.sh
```

### 第二步：创建 systemd 服务文件
创建服务文件（路径：`/etc/systemd/system/battery-threshold.service`）：
```bash
sudo nano /etc/systemd/system/battery-threshold.service
```

粘贴以下内容（优化依赖，确保系统就绪后执行）：
```ini
[Unit]
Description=Set Huawei laptop battery charge threshold to 80% on boot
Documentation=https://github.com/你的用户名/你的仓库名  # 替换为你的 GitHub 仓库地址
After=multi-user.target local-fs.target power.target
Requires=local-fs.target
DefaultDependencies=no

[Service]
Type=oneshot
ExecStart=/usr/local/bin/set_battery_threshold.sh
User=root
Group=root
RemainAfterExit=yes
# 输出日志到 systemd journal
StandardOutput=journal+console
StandardError=journal+console

[Install]
WantedBy=multi-user.target
```

### 第三步：启用并测试服务
```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启用服务（开机自启）
sudo systemctl enable battery-threshold.service

# 立即执行脚本（测试生效）
sudo systemctl start battery-threshold.service

# 检查服务状态（确认无报错）
sudo systemctl status battery-threshold.service

# 验证阈值是否生效
cat /sys/class/power_supply/BAT0/charge_control_start_threshold  # 应输出 70
cat /sys/class/power_supply/BAT0/charge_control_end_threshold    # 应输出 80
```

### 第四步：验证开机自启（可选）
重启笔记本后，执行以下命令验证：
```bash
cat /sys/class/power_supply/BAT0/charge_control_end_threshold
```
输出应为 `80`，说明自启生效。

## 额外操作

### 临时恢复 100% 充电（外出使用）
```bash
# 临时恢复满充（重启后自动回到 80%）
sudo sh -c "echo 95 > /sys/class/power_supply/BAT0/charge_control_start_threshold"
sudo sh -c "echo 100 > /sys/class/power_supply/BAT0/charge_control_end_threshold"
```

### 永久恢复 100% 充电
```bash
# 禁用并删除服务
sudo systemctl disable --now battery-threshold.service
sudo rm /etc/systemd/system/battery-threshold.service

# 删除脚本
sudo rm /usr/local/bin/set_battery_threshold.sh

# 重置阈值（可选）
sudo sh -c "echo 100 > /sys/class/power_supply/BAT0/charge_control_end_threshold"

# 重新加载 systemd 配置
sudo systemctl daemon-reload
```

## 故障排查
1. **阈值设置失败**：
   - 检查内核版本：`uname -r`（需 ≥ 5.15，华为笔记本 5.15 以上才支持阈值接口）；
   - 确认电池设备名：`ls /sys/class/power_supply/`（可能为 BAT1）；
   - 查看日志：`journalctl -u battery-threshold.service` 或 `cat /var/log/battery_threshold.log`。

2. **开机自启不生效**：
   - 检查服务是否启用：`sudo systemctl is-enabled battery-threshold.service`（应输出 `enabled`）；
   - 重新启用服务：`sudo systemctl reenable battery-threshold.service`。

3. **脚本权限错误**：
   - 重新添加权限：`sudo chmod 755 /usr/local/bin/set_battery_threshold.sh`。

## 总结
1. 轻量化方案：仅操作内核原生接口，无额外依赖，解决 TLP 发热问题；
2. 稳定性：systemd 服务确保开机必执行，日志可追溯；
3. 兼容性：适配华为笔记本 Arch Linux 环境，内核 ≥ 5.15 即可。

