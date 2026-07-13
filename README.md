# gpu\_monitor---yyzs

优云智算gpu\_monitor监控脚本

1.监控内核新增实时 GPU 利用率 / 显存打印，每次检测输出负载信息

2.所有日志、邮件内时间强制+8小时，解决容器实例 UTC 时区差 8 小时问题

3.保留原交互输入逻辑（邮箱 / 闲置时长 / 是否关机 / 关机等待）

4.保留开机自启脚本 /etc/profile.d/gpu\_monitor.sh

5.邮件正文附带触发瞬间 GPU 负载，便于排查

# 完整配置操作步骤

## 1.配置发件邮箱（必须先执行，否则邮件发送失败）

```bash
 vim /etc/msmtprc
 ```
写入以下模板（替换为你自己的发件邮箱信息）
```shell
[defaults
auth on
tls on
tls\_starttls on
tls\_certcheck off
host smtp.qq.com
port 587
from 你的发件邮箱@qq.com
user 你的发件邮箱@qq.com
password QQ邮箱授权码](bash)
```
加固文件权限（msmtp 强制要求）
运行
```bash
 chmod 600 /etc/msmtprc
 ```
## 2.保存部署脚本并执行
拉取脚本
```bash
wget https://github.com/gxy3366/gpu_monitor---yyzs/main/gpu_monitor.sh
```
```shell
#!/bin/bash
echo "===== 配置GPU空闲自动监控程序（邮件告警+实时负载打印+时区校正） ====="

# 1. 前置依赖检查安装
echo "1. 检查并安装依赖工具 msmtp bc"
if command -v apt &> /dev/null;then
    apt update -y
    apt install -y msmtp bc
elif command -v yum &> /dev/null;then
    yum install -y msmtp bc
fi

# 2. 交互式输入配置
## 输入告警邮箱
echo -n "请输入接收告警的邮箱地址:"
read mail_receiver

## 输入最大闲置时长（分钟）
while true; do
  echo -n "请输入GPU最大闲置时长（分钟）:"
  read max_idle
  if [[ "$max_idle" =~ ^[0-9]+$ && $max_idle -gt 0 ]]; then
    break
  else
    echo "输入必须为大于0的纯数字，请重新输入!"
  fi
done

## 是否开启自动关机
while true; do
    echo -n "闲置超时是否开启自动关机(y/n): "
    read shutdown_switch
    if [[ "$shutdown_switch" == "y" || "$shutdown_switch" == "n" ]];then
        break
    else
        echo "仅允许输入 y / n"
    fi
done

## 开启关机则输入关机等待时长
wait_time=0
if [ "$shutdown_switch" = "y" ]; then
    while true; do
      echo -n "请输入邮件通知后等待多少分钟关机:"
      read wait_time
      if [[ "$wait_time" =~ ^[0-9]+$ && $wait_time -gt 0 ]]; then
        break
      else
        echo "输入必须为大于0的纯数字，请重新输入!"
      fi
    done
fi

# 3. 写入改造后核心监控脚本到 /usr/local/bin/gpu_monitor
cat > /usr/local/bin/gpu_monitor << 'EOF'
#!/bin/bash
# GPU闲置监控核心程序（实时打印负载+时区校正+邮件告警）
LOG_FILE="/tmp/gpu_monitor.log"
# 内置硬件监控阈值
MONITOR_GPU_ID="0"
GPU_LOAD_THRESHOLD=5
MEM_LOAD_THRESHOLD=10
CHECK_INTERVAL=10

# 日志写入函数：时间强制+8小时修正时区偏差
write_log(){
    local now=$(date -d "+8 hour" "+%Y-%m-%d %H:%M:%S")
    echo "[$now] $1" >> $LOG_FILE
}

# 邮件发送函数
send_mail(){
    local receiver=$1
    local title="【GPU闲置告警】服务器即将自动关机"
    local content=$2
    echo -e "Subject:$title\nContent-Type:text/plain;charset=utf-8\n\n$content" | msmtp $receiver
    if [ $? -eq 0 ];then
        write_log "告警邮件发送成功"
    else
        write_log "邮件发送失败，请检查 /etc/msmtprc 邮箱配置"
    fi
}

# 检测GPU硬件是否存在
check_gpu(){
    if ! command -v nvidia-smi &> /dev/null;then
        write_log "无nvidia-smi，无GPU设备，监控退出"
        exit 0
    fi
    gpu_num=$(nvidia-smi --query-gpu=name --format=csv,noheader,nounits 2>/dev/null | wc -l)
    if [ $gpu_num -eq 0 ];then
        write_log "nvidia-smi可用，但未识别GPU硬件，监控退出"
        exit 0
    fi
    write_log "检测到GPU，总数量:$gpu_num，开始监控"
}

# 获取单张GPU利用率、显存利用率
get_gpu_util(){
    local gpu=$1
    gpu_u=$(nvidia-smi --query-gpu=utilization.gpu -i $gpu --format=csv,noheader,nounits 2>/dev/null)
    mem_u=$(nvidia-smi --query-gpu=utilization.memory -i $gpu --format=csv,noheader,nounits 2>/dev/null)
    echo "$gpu_u $mem_u"
}

# 拼接所有GPU实时负载信息字符串
get_all_load_info(){
    local load_info=""
    IFS=',' read -ra gpus <<< $MONITOR_GPU_ID
    for g in "${gpus[@]}";do
        read gu mu <<< $(get_gpu_util $g)
        load_info+="GPU${g}:利用率${gu}% 显存${mu}% | "
    done
    # 截断末尾多余分隔符
    echo "${load_info% | }"
}

# 判断所有监控GPU是否全部空闲
all_idle(){
    IFS=',' read -ra gpus <<< $MONITOR_GPU_ID
    for g in "${gpus[@]}";do
        read gu mu <<< $(get_gpu_util $g)
        if [ $gu -gt $GPU_LOAD_THRESHOLD ] || [ $mu -gt $MEM_LOAD_THRESHOLD ];then
            return 1 # 存在负载，非空闲
        fi
    done
    return 0
}

# 主监控循环
main(){
    # 读取外部传入参数
    MAX_IDLE=$1
    MAIL_TARGET=$2
    SHUTDOWN_FLAG=$3
    SHUTDOWN_WAIT=$4

    check_gpu
    write_log "================GPU监控服务启动================"
    idle_min=0
    while true;do
        # 获取当前全部GPU实时负载
        load_info=$(get_all_load_info)

        if all_idle;then
            idle_min=$(echo "scale=2;$idle_min + $CHECK_INTERVAL/60" | bc)
            write_log "GPU全部闲置，实时负载[$load_info]，累计${idle_min}min，阈值${MAX_IDLE}min"

            # 达到闲置阈值，发送邮件+延时关机
            if (( $(echo "$idle_min >= $MAX_IDLE" | bc -l) ));then
                server_ip=$(hostname -I | awk '{print $1}')
                mail_time=$(date -d "+8 hour" "+%Y-%m-%d %H:%M:%S")
                mail_body=$(cat <<MAIL
服务器IP: $server_ip
告警触发时间: $mail_time
累计闲置: ${idle_min}分钟
闲置阈值: ${MAX_IDLE}分钟
触发瞬间GPU负载: $load_info

取消自动关机命令：
shutdown -c
MAIL
)
                # 发送告警邮件
                send_mail $MAIL_TARGET "$mail_body"
                # 开启自动关机逻辑
                if [ "$SHUTDOWN_FLAG" = "y" ];then
                    write_log "邮件通知完成，${SHUTDOWN_WAIT}分钟后自动关机，执行 shutdown -c 取消关机"
                    shutdown -h +$SHUTDOWN_WAIT "GPU长期闲置自动关机"
                fi
                exit 0
            fi
        else
            # GPU存在负载，重置闲置计时器
            if (( $(echo "$idle_min > 0" | bc) ));then
                write_log "检测到GPU运行任务，实时负载[$load_info]，重置闲置计时器"
                idle_min=0
            else
                write_log "GPU存在运行任务，实时负载[$load_info]"
            fi
        fi
        sleep $CHECK_INTERVAL
    done
}

# 初始化日志文件
touch $LOG_FILE
chmod 644 $LOG_FILE
# 启动监控主程序
main $MAX_IDLE $MAIL_TARGET $SHUTDOWN_FLAG $SHUTDOWN_WAIT
EOF

# 赋予全局执行权限
chmod +x /usr/local/bin/gpu_monitor

# 4. 生成开机自启脚本 /etc/profile.d/gpu_monitor.sh
cat > /etc/profile.d/gpu_monitor.sh << EOF
#!/bin/bash
pid=\$(pgrep -f "/usr/local/bin/gpu_monitor")
if [ -z "\$pid" ]; then
  echo "启动GPU闲置监控程序"
  nohup /usr/local/bin/gpu_monitor $max_idle $mail_receiver $shutdown_switch $wait_time > /tmp/gpu_monitor.log 2>&1 &
else
  echo "GPU监控程序已在后台运行"
fi
EOF

# 自启脚本赋予执行权限
chmod +x /etc/profile.d/gpu_monitor.sh

# 5. 后台启动监控程序
echo "2. 后台启动GPU监控进程"
nohup /usr/local/bin/gpu_monitor $max_idle $mail_receiver $shutdown_switch $wait_time > /tmp/gpu_monitor.log 2>&1 &

# 6. 输出配置完成提示
echo -e "\n===== GPU监控配置完成 ====="
echo "闲置判定阈值：$max_idle 分钟"
if [ "$shutdown_switch" = "y" ];then
    echo "超时后邮件通知，等待 $wait_time 分钟自动关机"
fi
echo "告警接收邮箱：$mail_receiver"
echo "开机自启脚本路径：/etc/profile.d/gpu_monitor.sh"
echo "实时日志查看命令：tail -f /tmp/gpu_monitor.log"
echo "停止监控命令：pkill -f gpu_monitor"
echo "取消定时关机：shutdown -c"
echo -e "\n实时打印日志（Ctrl+C退出查看，不终止后台程序）"
sleep 1
tail -f /tmp/gpu_monitor.log
```
将上方完整代码复制，保存为 install\_gpu\_monitor.sh
添加执行权限
```bash
chmod +x install\_gpu\_monitor.sh
```
运行部署脚本，按提示交互输入配置
```bash
bash install\_gpu\_monitor.sh
 ```
 
交互输入项依次：  
接收告警邮箱  
GPU 最大闲置时长（分钟）  
是否开启自动关机 y/n  
若选 y，输入关机等待时长（分钟）  

## 3.日常命令

## 1.实时查看监控日志（可看到实时 GPU 负载、校正后的北京时间）
```bash
tail -f /tmp/gpu\_monitor.log
 ```
## 2.停止后台监控进程
```bash
pkill -f gpu\_monitor
 ```
## 3.取消已下发的定时关机
```bash
shutdown -c
 ```
## 4.重新手动启动监控（参数对应你部署时填写的配置）
```bash
# 格式：gpu\_monitor 闲置分钟 接收邮箱 是否关机(y/n) 关机等待分钟
nohup /usr/local/bin/gpu\_monitor 3 s202420211054@stu.tyust.edu.cn y 3 > /tmp/gpu\_monitor.log 2>&1 &
```

## 5.登录服务器自动加载开机自启
```bash
source /etc/profile.d/gpu\_monitor.sh
```
