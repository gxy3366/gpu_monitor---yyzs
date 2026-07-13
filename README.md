# gpu_monitor---yyzs
优云智算gpu_monitor监控脚本
1.监控内核新增实时 GPU 利用率 / 显存打印，每次检测输出负载信息
2.所有日志、邮件内时间强制+8小时，解决容器实例 UTC 时区差 8 小时问题
3.保留原交互输入逻辑（邮箱 / 闲置时长 / 是否关机 / 关机等待）
4.保留开机自启脚本 /etc/profile.d/gpu_monitor.sh
5.邮件正文附带触发瞬间 GPU 负载，便于排查
# 完整配置操作步骤
## 1.配置发件邮箱（必须先执行，否则邮件发送失败）
[vim /etc/msmtprc](shell)
写入以下模板（替换为你自己的发件邮箱信息）

[defaults
auth on
tls on
tls_starttls on
tls_certcheck off
host smtp.qq.com
port 587
from 你的发件邮箱@qq.com
user 你的发件邮箱@qq.com
password QQ邮箱授权码](bash)


加固文件权限（msmtp 强制要求）
运行[chmod 600 /etc/msmtprc](bash)
## 2.保存部署脚本并执行
将上方完整代码复制，保存为 install_gpu_monitor.sh
添加执行权限[chmod +x install_gpu_monitor.sh](bash)
运行部署脚本，按提示交互输入配置[bash install_gpu_monitor.sh](bash)


交互输入项依次：
接收告警邮箱
GPU 最大闲置时长（分钟）
是否开启自动关机 y/n
若选 y，输入关机等待时长（分钟）

## 3.日常命令
## 1.实时查看监控日志（可看到实时 GPU 负载、校正后的北京时间）
[tail -f /tmp/gpu_monitor.log](bash)
## 2.停止后台监控进程
[pkill -f gpu_monitor](bash)
## 3.取消已下发的定时关机
[shutdown -c](bash)
## 4.重新手动启动监控（参数对应你部署时填写的配置）

[# 格式：gpu_monitor 闲置分钟 接收邮箱 是否关机(y/n) 关机等待分钟
nohup /usr/local/bin/gpu_monitor 3 s202420211054@stu.tyust.edu.cn y 3 > /tmp/gpu_monitor.log 2>&1 &](bash)
## 5.登录服务器自动加载开机自启
[source /etc/profile.d/gpu_monitor.sh](bash)

