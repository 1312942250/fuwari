---
title: Minio客户端mc mrrior用法
published: 2025-12-04
description: '介绍Minio客户端mc mrrior在Linux中的使用，包括安装、配置、编写脚本。'
image: ''
tags: ['计算机']
category: '学习'
draft: false 
lang: ''
---

> 参考[官方文档](https://minio.org.cn/docs/minio/linux/reference/minio-mc/mc-mirror.html)

## 下载

```sh
# 下载mc客户端
wget https://dl.min.io/client/mc/release/linux-amd64/mc
# 赋予权限
chmod +x mc
# 迁移到/usr目录
sudo mv mc /usr/local/bin/
```

## 命令介绍

1、`mc mirror`用来同步当前服务器的minio数据到远程服务器。

常用参数：

- `--watch` 使mc mirror处理持续监控的状态，即源对象添加或者删除文件，目标对象会实时同步操作。
- `--remove` 删除目标上不存在于源上的对象。
- `--overwrite` 覆盖目标对象已存在的数据。
- `--limit-download` 限制目标对象的下载速度。
- `--limit-upload` 限制源对象的上传速度。

2、`mc alias set`为对象设置别名

3、`mc du`查看指定对象的文件大小及数量

4、`mc diff`查看两个对象的文件差异

## 监控脚本

```sh
#!/bin/bash

# ==============================================================
# MinIO 实时同步监控服务脚本 (Watch Mode - 无自动重启)
#
# 功能: 1. 以 --watch 模式监控并实时同步指定的桶
#       2. 日志按天切割，自动保留30天
#       3. 提供 start|stop|status|restart 服务管理功能
#       4. 进程异常退出时不自动重启
# ==============================================================

# ================= 1. 服务配置 =================
SRC_NAME="minio_src"
SRC_URL="http://192.168.227.128:9000"
SRC_ACCESS="minioadmin"
SRC_SECRET="minioadmin"

DST_NAME="minio_dst"
DST_URL="http://192.168.227.129:9000"
DST_ACCESS="minioadmin"
DST_SECRET="minioadmin"

# ********************** 重点配置 **********************
# 要监控的桶列表（支持通配符 * 表示所有桶）
# 示例: MONITOR_BUCKETS=("ruoyi" "test-bucket" "archive-*")
MONITOR_BUCKETS=("*")  # 在此处修改为您要监控的桶名数组
# ***************************************************

# 日志目录配置
LOG_DIR="./watch_log"
LOG_BASENAME="minio_watch_only"
LOG_EXT=".log"

# Watch 模式同步参数
OVERWRITE_FLAG="--overwrite"
# 注意: 实时同步通常不加 --remove，避免误删目标端手动添加的文件
# 如需完全镜像，可添加: MIRROR_FLAGS="$OVERWRITE_FLAG --remove"
MIRROR_FLAGS="$OVERWRITE_FLAG"

# 可选速率限制
UPLOAD_RATE_LIMIT="25M"    # 例如: "10M"
DOWNLOAD_RATE_LIMIT="15M"  # 例如: "10M"

# 进程标识文件
PID_DIR="./.pids"
# ==============================================

# 设置语言保证日志日期为英文/数字格式
export LANG=C

# 0. 初始化函数
init_dirs() {
    mkdir -p "$LOG_DIR"
    mkdir -p "$PID_DIR"
    chmod 755 "$LOG_DIR" "$PID_DIR" 2>/dev/null
}

# 1. 日志函数 - 按天滚动
setup_logging() {
    TODAY=$(date +%Y%m%d)
    CURRENT_LOG="${LOG_DIR}/${LOG_BASENAME}_${TODAY}${LOG_EXT}"
    
    if [ ! -f "$CURRENT_LOG" ]; then
        echo "=== MinIO Watch-Only Sync Log - Started: $(date '+%Y-%m-%d %H:%M:%S') ===" > "$CURRENT_LOG"
        echo "Source: $SRC_NAME ($SRC_URL)" >> "$CURRENT_LOG"
        echo "Target: $DST_NAME ($DST_URL)" >> "$CURRENT_LOG"
        echo "Sync Flags: $MIRROR_FLAGS" >> "$CURRENT_LOG"
        echo "Mode: Real-time watch only (NO initial full sync)" >> "$CURRENT_LOG"
        echo "==================================================" >> "$CURRENT_LOG"
    fi
}

# 2. 清理旧日志 (保留30天)
cleanup_old_logs() {
    find "$LOG_DIR" -name "${LOG_BASENAME}_*${LOG_EXT}" -type f -mtime +30 -delete 2>/dev/null
    echo "[$(date '+%H:%M:%S')] 旧日志清理完成。" >> "$CURRENT_LOG"
}

# 3. 速率限制标志构建
build_rate_flags() {
    local rate_flags=""
    if [ -n "$UPLOAD_RATE_LIMIT" ]; then
        rate_flags="$rate_flags --limit-upload $UPLOAD_RATE_LIMIT"
    fi
    if [ -n "$DOWNLOAD_RATE_LIMIT" ]; then
        rate_flags="$rate_flags --limit-download $DOWNLOAD_RATE_LIMIT"
    fi
    echo "$rate_flags"
}

# 4. 获取最终要监控的桶列表
get_final_bucket_list() {
    local cli_bucket="$1"
    local final_list=""
    
    # 优先级1: 命令行参数
    if [ -n "$cli_bucket" ]; then
        final_list="$cli_bucket"
        echo "[$(date '+%H:%M:%S')] 命令行指定监控桶: $final_list" >> "$CURRENT_LOG"
        echo "$final_list"
        return 0
    fi
    
    # 优先级2: 脚本内置的 MONITOR_BUCKETS 数组
    if [ ${#MONITOR_BUCKETS[@]} -gt 0 ]; then
        # 检查通配符 "*"
        for bucket_pattern in "${MONITOR_BUCKETS[@]}"; do
            if [ "$bucket_pattern" = "*" ]; then
                echo "[$(date '+%H:%M:%S')] 配置为监控所有桶，正在扫描..." >> "$CURRENT_LOG"
                final_list=$(mc ls "$SRC_NAME" 2>/dev/null | awk '{print $NF}' | grep '/$' | sed 's/\///g')
                if [ -z "$final_list" ]; then
                    echo "[$(date '+%H:%M:%S')] 错误: 未在源服务器上发现任何桶" >> "$CURRENT_LOG"
                    return 1
                fi
                echo "[$(date '+%H:%M:%S')] 发现所有桶: $final_list" >> "$CURRENT_LOG"
                echo "$final_list"
                return 0
            fi
        done
        
        # 使用数组中的具体桶名
        final_list="${MONITOR_BUCKETS[*]}"
        echo "[$(date '+%H:%M:%S')] 脚本内置监控桶列表: $final_list" >> "$CURRENT_LOG"
        
        # 验证桶是否存在
        local existing_buckets=""
        for bucket in $final_list; do
            if mc ls "$SRC_NAME/$bucket" >/dev/null 2>&1; then
                existing_buckets="$existing_buckets $bucket"
            else
                echo "[$(date '+%H:%M:%S')] 警告: 桶 [$bucket] 不存在，已跳过" >> "$CURRENT_LOG"
            fi
        done
        
        final_list=$(echo "$existing_buckets" | sed 's/^ //')
        if [ -z "$final_list" ]; then
            echo "[$(date '+%H:%M:%S')] 错误: 所有指定桶均不存在" >> "$CURRENT_LOG"
            return 1
        fi
        
        echo "$final_list"
        return 0
    fi
    
    # 优先级3: 默认监控所有桶
    echo "[$(date '+%H:%M:%S')] 未指定监控桶，默认监控所有桶..." >> "$CURRENT_LOG"
    final_list=$(mc ls "$SRC_NAME" 2>/dev/null | awk '{print $NF}' | grep '/$' | sed 's/\///g')
    if [ -z "$final_list" ]; then
        echo "[$(date '+%H:%M:%S')] 错误: 未在源服务器上发现任何桶" >> "$CURRENT_LOG"
        return 1
    fi
    echo "[$(date '+%H:%M:%S')] 发现所有桶: $final_list" >> "$CURRENT_LOG"
    echo "$final_list"
}

# 5. 启动单个桶的监控进程（仅监控，无全量同步）
start_bucket_watch() {
    local bucket="$1"
    local pid_file="${PID_DIR}/watch_${bucket}.pid"
    
    # 检查是否已运行
    if [ -f "$pid_file" ]; then
        local pid=$(cat "$pid_file" 2>/dev/null)
        if kill -0 "$pid" 2>/dev/null; then
            echo "[$(date '+%H:%M:%S')] 桶 [$bucket] 监控已在运行 (PID: $pid)" >> "$CURRENT_LOG"
            return 0
        else
            rm -f "$pid_file"
            echo "[$(date '+%H:%M:%S')] 清理残留PID文件: $pid_file" >> "$CURRENT_LOG"
        fi
    fi
    
    # 启动监控进程（仅 --watch 模式）
    {
        echo "[$(date '+%H:%M:%S')] 启动桶 [$bucket] 的实时监控进程..." >> "$CURRENT_LOG"
        
        # 直接进入实时监控模式
        mc mirror --watch \
            "$SRC_NAME/$bucket" "$DST_NAME/$bucket" \
            $MIRROR_FLAGS \
            $(build_rate_flags) \
            >> "$CURRENT_LOG" 2>&1
        
        # 执行到这里说明 mc mirror --watch 进程已退出
        local exit_time=$(date '+%H:%M:%S')
        echo "[$exit_time] 监控进程 [$bucket] 已退出。" >> "$CURRENT_LOG"
        
        # 清理PID文件
        rm -f "$pid_file"
        echo "[$exit_time] 已清理PID文件: $pid_file" >> "$CURRENT_LOG"
        
    } &
    
    local watch_pid=$!
    echo $watch_pid > "$pid_file"
    echo "[$(date '+%H:%M:%S')] 桶 [$bucket] 监控进程已启动 (PID: $watch_pid)" >> "$CURRENT_LOG"
    echo "[$(date '+%H:%M:%S')] PID文件: $pid_file" >> "$CURRENT_LOG"
}

# 6. 停止服务函数（使用 pkill 增强）
stop_service() {
    echo "[$(date '+%H:%M:%S')] 停止监控服务..." | tee -a "$CURRENT_LOG"
    
    # 方法1: 通过PID文件尝试停止
    local pids=$(ls ${PID_DIR}/watch_*.pid 2>/dev/null 2>/dev/null)
    if [ -n "$pids" ]; then
        for pid_file in $pids; do
            if [ -f "$pid_file" ]; then
                local pid=$(cat "$pid_file" 2>/dev/null)
                local bucket=$(basename "$pid_file" | sed 's/^watch_//' | sed 's/\.pid$//')
                
                if [ -n "$pid" ] && kill -0 "$pid" 2>/dev/null; then
                    echo "[$(date '+%H:%M:%S')] 停止桶 [$bucket] 监控 (PID: $pid)..." >> "$CURRENT_LOG"
                    kill -TERM "$pid" 2>/dev/null
                    sleep 1
                fi
                rm -f "$pid_file"
            fi
        done
    fi
    
    # 方法2: 使用 pkill 强制清理所有相关进程（确保停止）
    echo "[$(date '+%H:%M:%S')] 使用 pkill 清理残留进程..." >> "$CURRENT_LOG"
    
    # 精准匹配模式
    local pattern="mc mirror.*(minio_src|minio_dst|${SRC_URL//http:\/\//}|${DST_URL//http:\/\//})"
    
    # 先尝试优雅终止
    pkill -TERM -f "$pattern" 2>/dev/null
    sleep 2
    
    # 强制终止任何残留
    if pgrep -f "$pattern" > /dev/null; then
        echo "[$(date '+%H:%M:%S')] 强制终止残留进程..." >> "$CURRENT_LOG"
        pkill -KILL -f "$pattern" 2>/dev/null
    fi
    
    # 最终确认
    sleep 1
    if pgrep -f "$pattern" > /dev/null; then
        echo "[$(date '+%H:%M:%S')] ⚠️  仍有进程残留，请手动检查。" | tee -a "$CURRENT_LOG"
    else
        echo "[$(date '+%H:%M:%S')] ✅ 所有监控进程已停止。" | tee -a "$CURRENT_LOG"
    fi
}

# 7. 服务状态检查
check_status() {
    echo "=== MinIO 监控服务状态 (纯监控模式) ==="
    echo "当前时间: $(date '+%Y-%m-%d %H:%M:%S')"
    echo "日志目录: $LOG_DIR (保留30天)"
    echo "PID 目录: $PID_DIR"
    echo "内置监控桶: ${MONITOR_BUCKETS[*]:-未配置}"
    echo ""
    
    local pids=$(ls ${PID_DIR}/watch_*.pid 2>/dev/null 2>/dev/null | sort)
    
    if [ -z "$pids" ]; then
        echo "状态: 服务未运行"
        return 1
    fi
    
    echo "运行中的监控进程:"
    echo "桶名称                 PID      状态"
    echo "-------------------  --------  ---------"
    
    local running_count=0
    for pid_file in $pids; do
        if [ -f "$pid_file" ]; then
            local pid=$(cat "$pid_file" 2>/dev/null)
            local bucket=$(basename "$pid_file" | sed 's/^watch_//' | sed 's/\.pid$//')
            local status="UNKNOWN"
            
            if [ -n "$pid" ]; then
                if kill -0 "$pid" 2>/dev/null; then
                    status="🟢 RUNNING"
                    running_count=$((running_count + 1))
                else
                    status="🔴 DEAD"
                fi
            else
                status="⚪ NO_PID"
            fi
            
            printf "%-20s %-9s %-10s\n" "$bucket" "$pid" "$status"
        fi
    done
    
    echo ""
    echo "统计: $running_count 个进程运行中"
    echo "今日日志: $CURRENT_LOG"
    
    if [ $running_count -eq 0 ]; then
        echo "总体状态: ⭕ 无活跃进程"
        return 2
    else
        echo "总体状态: ✅ 服务运行中"
        return 0
    fi
}

# 8. 主服务启动函数
start_service() {
    local cli_bucket="$1"
    
    # 必须先初始化日志系统
    init_dirs
    setup_logging
    cleanup_old_logs
    
    echo "启动 MinIO 纯实时监控服务..." | tee -a "$CURRENT_LOG"
    echo "[$(date '+%H:%M:%S')] 服务启动..." >> "$CURRENT_LOG"
    
    # 检查 mc 命令
    if ! command -v mc &> /dev/null; then
        echo "错误: 未找到 'mc' 命令" | tee -a "$CURRENT_LOG"
        exit 1
    fi
    
    # 设置 MinIO 别名
    echo "[$(date '+%H:%M:%S')] 设置 MinIO 服务器别名..." >> "$CURRENT_LOG"
    mc alias set "$SRC_NAME" "$SRC_URL" "$SRC_ACCESS" "$SRC_SECRET" --api S3v4 >> "$CURRENT_LOG" 2>&1
    mc alias set "$DST_NAME" "$DST_URL" "$DST_ACCESS" "$DST_SECRET" --api S3v4 >> "$CURRENT_LOG" 2>&1
    
    # 获取最终要监控的桶列表
    echo "[$(date '+%H:%M:%S')] 确定监控桶列表..." >> "$CURRENT_LOG"
    BUCKET_LIST=$(get_final_bucket_list "$cli_bucket")
    
    if [ $? -ne 0 ] || [ -z "$BUCKET_LIST" ]; then
        echo "错误: 无法获取有效的桶列表，服务启动失败" | tee -a "$CURRENT_LOG"
        exit 1
    fi
    
    echo "最终监控桶列表: $BUCKET_LIST" | tee -a "$CURRENT_LOG"
    
    # 启动每个桶的监控
    for BUCKET in $BUCKET_LIST; do
        start_bucket_watch "$BUCKET"
        sleep 1  # 避免同时启动所有进程
    done
    
    echo "MinIO 纯实时监控服务已启动完成" | tee -a "$CURRENT_LOG"
    echo "监控模式: --watch only (无初始全量同步)" | tee -a "$CURRENT_LOG"
    echo "监控桶列表: $BUCKET_LIST" | tee -a "$CURRENT_LOG"
    echo "日志文件: $CURRENT_LOG" | tee -a "$CURRENT_LOG"
    echo "停止服务: $0 stop" | tee -a "$CURRENT_LOG"
    
    echo "[$(date '+%H:%M:%S')] 服务启动流程完成" >> "$CURRENT_LOG"
}

# 9. 主控制流程（修正了函数调用顺序）
case "${1:-}" in
    start)
        # 注意：start_service 内部已初始化日志
        start_service "${2:-}"
        ;;
    stop)
        init_dirs
        setup_logging
        stop_service
        ;;
    status)
        init_dirs
        setup_logging
        check_status
        ;;
    restart)
        init_dirs
        setup_logging
        stop_service
        sleep 2
        start_service "${2:-}"
        ;;
    *)
        echo "使用方法: $0 {start|stop|status|restart} [单个桶名]"
        echo ""
        echo "示例:"
        echo "  $0 start                   # 使用脚本内置 MONITOR_BUCKETS 配置启动"
        echo "  $0 start ruoyi             # 只监控 ruoyi 桶（覆盖内置配置）"
        echo "  $0 start \"bucket1 bucket2\" # 监控多个桶"
        echo "  $0 stop                    # 停止所有监控进程"
        echo "  $0 status                  # 查看服务状态"
        echo "  $0 restart                 # 重启服务"
        echo ""
        echo "配置说明:"
        echo "  1. 内置桶配置: 编辑脚本顶部的 MONITOR_BUCKETS 数组"
        echo "  2. 日志目录: $LOG_DIR/${LOG_BASENAME}_YYYYMMDD.log"
        echo "  3. 自动保留: 30天"
        echo "  4. 模式: 纯实时监控 (--watch only)，不做初始全量同步"
        echo ""
        echo "当前内置监控桶配置: ${MONITOR_BUCKETS[*]:-未配置}"
        exit 1
        ;;
esac
```

## 全量备份脚本

```sh
#!/bin/bash

# ==============================================================
# MinIO 全局同步脚本 (自动多桶处理与可选速率限制)
#
# 功能: 1. 可选指定桶，否则同步源服务器上的所有桶。
#       2. 可选限制上传和下载速率。
#       3. 按桶进行自动分批处理。
# ==============================================================

# ================= 1. MinIO 连接配置 =================
SRC_NAME="minio_src"
SRC_URL="http://192.168.227.128:9000"
SRC_ACCESS="minioadmin"
SRC_SECRET="minioadmin"

DST_NAME="minio_dst"
DST_URL="http://192.168.227.129:9000"
DST_ACCESS="minioadmin"
DST_SECRET="minioadmin"

LOG_FILE="./minio_global_sync_$(date +%Y%m%d_%H%M%S).log"
# ====================================================

# 0. 参数解析与默认值设置
# 参数顺序: [桶名称] [上传速率] [下载速率]
BUCKET_NAME_PARAM=${1:-""}      # 第 1 个参数，默认为空 (表示同步所有桶)
UPLOAD_RATE_LIMIT=${2:-""}      # 第 2 个参数，默认为空 (表示不限制)
DOWNLOAD_RATE_LIMIT=${3:-""}    # 第 3 个参数，默认为空 (表示不限制)

# 动态构造上传和下载限制标志
UPLOAD_FLAG=""
if [ -n "$UPLOAD_RATE_LIMIT" ]; then
    UPLOAD_FLAG="--limit-upload $UPLOAD_RATE_LIMIT"
fi

DOWNLOAD_FLAG=""
if [ -n "$DOWNLOAD_RATE_LIMIT" ]; then
    DOWNLOAD_FLAG="--limit-download $DOWNLOAD_RATE_LIMIT"
fi

RATE_LIMIT_FLAGS="$UPLOAD_FLAG $DOWNLOAD_FLAG"

echo ">> MinIO 全局同步任务开始时间：$(date)" >> "$LOG_FILE"
echo ">> 上传速率限制设置为: ${UPLOAD_RATE_LIMIT:-无限制}" >> "$LOG_FILE"
echo ">> 下载速率限制设置为: ${DOWNLOAD_RATE_LIMIT:-无限制}" >> "$LOG_FILE"


# 1. 环境检查与别名设置 (保持不变)
if ! command -v mc &> /dev/null; then 
    echo "错误: 未找到 'mc' 命令。请先安装 MinIO Client。" >> "$LOG_FILE"
    exit 1
fi
echo "正在设置服务器别名..." >> "$LOG_FILE"
mc alias set "$SRC_NAME" "$SRC_URL" "$SRC_ACCESS" "$SRC_SECRET" --api S3v4 >> "$LOG_FILE" 2>&1
mc alias set "$DST_NAME" "$DST_URL" "$DST_ACCESS" "$DST_SECRET" --api S3v4 >> "$LOG_FILE" 2>&1

# 2. 桶发现与目标列表生成
TARGET_BUCKETS=""

if [ -n "$BUCKET_NAME_PARAM" ]; then
    # 传入了桶名，只同步指定的桶
    TARGET_BUCKETS="$BUCKET_NAME_PARAM"
    echo ">> 任务目标: 只同步指定桶 [${BUCKET_NAME_PARAM}]" >> "$LOG_FILE"
else
    # 未传入桶名，列出源服务器上的所有桶
    echo ">> 任务目标: 扫描源服务器 [$SRC_NAME] 上的所有桶..." >> "$LOG_FILE"
    # mc ls 会列出所有桶名，并以 / 结尾。awk 和 sed 用于清理输出。
    TARGET_BUCKETS=$(mc ls "$SRC_NAME" 2>/dev/null | awk '{print $NF}' | grep '/$' | sed 's/\///g')
    
    if [ -z "$TARGET_BUCKETS" ]; then
        echo "警告: 未在源服务器上发现任何桶，任务终止。" >> "$LOG_FILE"
        exit 0
    fi
    echo ">> 发现以下桶进行同步: ${TARGET_BUCKETS}" >> "$LOG_FILE"
fi

# 3. 循环同步每个桶 (自动分批处理)
TOTAL_BUCKETS=$(echo "$TARGET_BUCKETS" | wc -w)
SUCCESS_COUNT=0
FAILURE_COUNT=0

for BUCKET in $TARGET_BUCKETS; do
    echo "--- 正在同步整个桶 [${BUCKET}] ---" >> "$LOG_FILE"
    
    SRC_PATH="$SRC_NAME/$BUCKET"
    DST_PATH="$DST_NAME/$BUCKET"

    # 核心命令: 同步整个桶。mc mirror 内部会处理高并发文件传输。
    mc mirror "$SRC_PATH" "$DST_PATH" \
      --overwrite \
      $RATE_LIMIT_FLAGS \
      >> "$LOG_FILE" 2>&1

    # 检查单次同步结果
    if [ $? -eq 0 ]; then
        echo "✅ 桶 [${BUCKET}] 同步成功。" >> "$LOG_FILE"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    else
        echo "❌ 桶 [${BUCKET}] 同步失败，请检查日志中的具体错误。" >> "$LOG_FILE"
        FAILURE_COUNT=$((FAILURE_COUNT + 1))
    fi
    echo "" >> "$LOG_FILE"
done

# 4. 任务总结 
echo "================ 任务总结 ================" >> "$LOG_FILE"
echo "总共处理桶数: $TOTAL_BUCKETS" >> "$LOG_FILE"
echo "成功同步桶数: $SUCCESS_COUNT" >> "$LOG_FILE"
echo "失败同步桶数: $FAILURE_COUNT" >> "$LOG_FILE"

if [ $FAILURE_COUNT -eq 0 ]; then
    echo "✅ 所有桶同步完成，全部成功！" >> "$LOG_FILE"
    exit 0
else
    echo "❌ 任务结束，有桶同步失败，请检查日志。" >> "$LOG_FILE"
    exit 1
fi
echo "任务结束时间：$(date)" >> "$LOG_FILE"
```

## 按文件夹分批备份脚本

```sh
#!/bin/bash

# ==============================================================
# MinIO 逐日分批同步脚本 (参数化速率限制 - 带详细日志)
# ==============================================================

# ================= 1. MinIO 连接配置 =================
SRC_NAME="minio_src"
SRC_URL="http://192.168.227.128:9000"
SRC_ACCESS="minioadmin"
SRC_SECRET="minioadmin"

DST_NAME="minio_dst"
DST_URL="http://192.168.227.129:9000"
DST_ACCESS="minioadmin"
DST_SECRET="minioadmin"

LOG_FILE="./minio_sync_$(date +%Y%m%d_%H%M%S).log"
# ====================================================

# 0. 参数解析与校验 
if [ $# -lt 2 ]; then
    echo "错误：参数不足。"
    echo "使用方法: $0 <Bucket名称> <月份路径前缀> [上传速率限制] [下载速率限制]"
    echo "示例 1 (不限制速率): $0 archive 2025/11"
    echo "示例 2 (限制上传速率为 10 MiB/s): $0 archive 2025/11 10MiB"
    exit 1
fi

BUCKET_NAME=$1
MONTH_PREFIX=$2 

# 使用 Shell 参数扩展设置可选参数的默认值 (如果未提供，则为空字符串)
UPLOAD_RATE_LIMIT=${3:-""}
DOWNLOAD_RATE_LIMIT=${4:-""}

# 动态构造上传和下载限制标志
UPLOAD_FLAG=""
if [ -n "$UPLOAD_RATE_LIMIT" ]; then
    UPLOAD_FLAG="--limit-upload $UPLOAD_RATE_LIMIT"
fi

DOWNLOAD_FLAG=""
if [ -n "$DOWNLOAD_RATE_LIMIT" ]; then
    DOWNLOAD_FLAG="--limit-download $DOWNLOAD_RATE_LIMIT"
fi

RATE_LIMIT_FLAGS="$UPLOAD_FLAG $DOWNLOAD_FLAG"

echo ">> MinIO 逐日分批同步任务开始时间：$(date)" >> "$LOG_FILE"
echo ">> 本次同步范围: Bucket=[$BUCKET_NAME], 月份路径=[$MONTH_PREFIX]" >> "$LOG_FILE"
echo ">> 上传速率限制设置为: ${UPLOAD_RATE_LIMIT:-无限制}" >> "$LOG_FILE"
echo ">> 下载速率限制设置为: ${DOWNLOAD_RATE_LIMIT:-无限制}" >> "$LOG_FILE"


# 1. 环境检查与别名设置 (保持不变)
if ! command -v mc &> /dev/null; then 
    echo "错误: 未找到 'mc' 命令。请先安装 MinIO Client。" >> "$LOG_FILE"
    exit 1
fi
echo "正在设置服务器别名..." >> "$LOG_FILE"
mc alias set "$SRC_NAME" "$SRC_URL" "$SRC_ACCESS" "$SRC_SECRET" --api S3v4 >> "$LOG_FILE" 2>&1
mc alias set "$DST_NAME" "$DST_URL" "$DST_ACCESS" "$DST_SECRET" --api S3v4 >> "$LOG_FILE" 2>&1

# 2. 扫描并发现日文件夹 (保持不变)
FULL_MONTH_PATH="$SRC_NAME/$BUCKET_NAME/$MONTH_PREFIX"
echo "正在扫描源路径 [$FULL_MONTH_PATH] 下的日文件夹..." >> "$LOG_FILE"

DAY_FOLDERS=$(mc ls "$FULL_MONTH_PATH/" 2>/dev/null | awk '{print $NF}' | grep '/$' | sed 's/\///g')

if [ -z "$DAY_FOLDERS" ]; then
    echo "警告: 在 [$FULL_MONTH_PATH] 下未找到任何日文件夹（如 01, 02 等）。请检查路径是否正确或文件夹是否存在。" >> "$LOG_FILE"
    SYNC_STATUS=0
    TOTAL_FAILURES=0
else
    echo "成功发现以下日文件夹，开始逐个同步: $DAY_FOLDERS" >> "$LOG_FILE"

    # 3. 循环同步每个日文件夹
    TOTAL_FAILURES=0
    for DAY in $DAY_FOLDERS; do
        FULL_DAY_PREFIX="$MONTH_PREFIX/$DAY"
        
        SRC_PATH="$SRC_NAME/$BUCKET_NAME/$FULL_DAY_PREFIX"
        DST_PATH="$DST_NAME/$BUCKET_NAME/$FULL_DAY_PREFIX"

        echo "--- 正在同步文件夹 [$FULL_DAY_PREFIX] ---" >> "$LOG_FILE"

        # 核心命令: 恢复所有日志输出 (stdout 和 stderr) 到日志文件
        mc mirror "$SRC_PATH" "$DST_PATH" \
          --overwrite \
          $RATE_LIMIT_FLAGS \
          >> "$LOG_FILE" 2>&1 # <--- 关键修改

        # 检查单次同步结果
        if [ $? -eq 0 ]; then
            echo "✅ 文件夹 [$FULL_DAY_PREFIX] 同步成功。" >> "$LOG_FILE"
        else
            echo "❌ 文件夹 [$FULL_DAY_PREFIX] 同步失败，请检查日志中的具体错误。" >> "$LOG_FILE"
            TOTAL_FAILURES=$((TOTAL_FAILURES + 1))
        fi
        echo "" >> "$LOG_FILE"
    done
fi

# 4. 任务总结 
echo "================ 任务总结 ================" >> "$LOG_FILE"
if [ $TOTAL_FAILURES -eq 0 ]; then
    echo "✅ 所有日文件夹同步完成，全部成功！" >> "$LOG_FILE"
    SYNC_STATUS=0
else
    echo "❌ 任务结束，共有 $TOTAL_FAILURES 个日文件夹同步失败，请检查日志。" >> "$LOG_FILE"
    SYNC_STATUS=1
fi
echo "任务结束时间：$(date)" >> "$LOG_FILE"
exit $SYNC_STATUS
```









