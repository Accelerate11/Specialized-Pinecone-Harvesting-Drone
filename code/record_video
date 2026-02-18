!/bin/bash
# 自动网络配置与视频录制脚本

# 配置参数
CAMERA_IP="192.168.144.25"        # A8 mini默认IP（文档第4.9节）
JETSON_IP="192.168.144.30"        # Jetson静态IP（与相机同网段）
INTERFACE="eth0"                  # 网络接口名称，根据实际修改
RTSP_URL="rtsp://$CAMERA_IP:8554/main.264"  # A8 mini默认RTSP地址（文档第4.4节）
OUTPUT_DIR="/home/nvidia/Videos"  # 视频保存目录

# 生成带时间戳的文件名
FILENAME="a8mini_recording_$(date +%Y%m%d_%H%M%S).mp4"
OUTPUT_PATH="$OUTPUT_DIR/$FILENAME"

# 函数：记录日志
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# 1. 检查网络接口是否存在
log "检查网络接口 $INTERFACE..."
if ! ip link show "$INTERFACE" &>/dev/null; then
    log "错误: 接口 $INTERFACE 不存在。请检查接口名称（如 eth0、enp0s1）。"
    exit 1
fi

# 2. 清除接口现有IP设置（避免冲突）
log "清除接口 $INTERFACE 的旧IP设置..."
sudo ip addr flush dev "$INTERFACE" 2>/dev/null

# 3. 设置静态IP
log "设置接口 $INTERFACE 的IP为 $JETSON_IP/24..."
sudo ip addr add "$JETSON_IP/24" dev "$INTERFACE"
if [ $? -ne 0 ]; then
    log "警告: IP设置失败，可能已存在相同IP。尝试继续运行。"
fi

# 4. 启用接口
sudo ip link set dev "$INTERFACE" up
sleep 2  # 等待接口稳定

# 5. 验证网络连通性
log "检查相机连通性（ping $CAMERA_IP）..."
if ping -c 3 -W 3 "$CAMERA_IP" &>/dev/null; then
    log "网络连通性正常。"
else
    log "警告: 无法ping通相机。请检查相机供电、网线连接。"
    # 不退出，尝试继续录制（可能相机需更长时间启动）
fi

# 6. 检查FFmpeg依赖
if ! command -v ffmpeg &>/dev/null; then
    log "错误: FFmpeg未安装。正在尝试安装..."
    sudo apt update && sudo apt install -y ffmpeg
    if [ $? -ne 0 ]; then
        log "错误: FFmpeg安装失败。请手动运行: sudo apt install ffmpeg"
        exit 1
    fi
fi

# 7. 开始录制视频
log "开始录制视频..."
log "相机RTSP流: $RTSP_URL"
log "输出文件: $OUTPUT_PATH"
log "按 Ctrl+C 停止录制。"

# 使用FFmpeg录制（-c copy 直接复制流，减少延迟）
ffmpeg -i "$RTSP_URL" -c copy -f mp4 -y "$OUTPUT_PATH"
ffmpeg_exit_code=$?

# 录制结束处理
if [ $ffmpeg_exit_code -eq 0 ]; then
    log "录制已停止。视频已保存到: $OUTPUT_PATH"
elif [ $ffmpeg_exit_code -eq 255 ]; then
    log "录制已被用户终止。视频已保存到: $OUTPUT_PATH"
    # 检查文件是否存在且非空
    if [ -s "$OUTPUT_PATH" ]; then
        log "录制文件大小: $(du -h "$OUTPUT_PATH" | cut -f1)"
    else
        log "警告: 录制文件为空或不存在。"
    fi
else
    log "录制过程异常 (退出码: $ffmpeg_exit_code)。"
    if [ -f "$OUTPUT_PATH" ]; then
        if [ -s "$OUTPUT_PATH" ]; then
            log "警告: 录制文件已保存但可能有损坏: $OUTPUT_PATH"
            log "文件大小: $(du -h "$OUTPUT_PATH" | cut -f1)"
        else
            log "错误: 录制文件为空，正在删除..."
            rm -f "$OUTPUT_PATH"
        fi
    fi
fi    
exit $ffmpeg_exit_code
                                                              
