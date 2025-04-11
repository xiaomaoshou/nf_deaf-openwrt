**一、编译方法 以immortalwrt-mt798x 为例：**
1. 准备工作
   
    1.1 克隆 immortalwrt-mt798x 仓库
    git clone https://github.com/hanwckf/immortalwrt-mt798x.git
    cd immortalwrt-mt798x
   
    1.2 更新 feeds 并安装依赖
   
    ./scripts/feeds update -a
   
    ./scripts/feeds install -a
    
    1.3 添加 nf_deaf 源码
    git clone https://github.com/kob/nf_deaf-openwrt.git package/kernel/nf_deaf
    
2. 配置编译选项

   2.1 启用 nf_deaf 模块
    运行菜单配置工具：
    make menuconfig
    导航到以下路径并启用模块：
    Kernel Modules → Netfilter Extensions → <*> kmod-nf-deaf
   
   2.2 确认依赖模块已启用
    确保 kmod-nf-conntrack 已启用（路径同上）。    
4. 编译完整固件
    make -j$(nproc) V=s
   
5. 仅编译 nf_deaf 模块
   make package/kernel/nf_deaf/compile V=s
   编译结果位于bin/targets/<架构>/packages/kmod-nf-deaf_*.ipk
备注：需要修改伪造请求内容请见https://github.com/kob/nf_deaf


**二、使用方法**
1. 加载内核模块：insmod nf_deaf.ko 
检查是否加载成功：
dmesg | tail  # 查看内核日志，确认无错误
lsmod | grep nf_deaf  # 确认模块已加载
2. 配置数据包标记（skb->mark）
模块通过 skb->mark 的特定位域触发逻辑。需使用 iptables 或 nftables 设置标记。
示例：标记 TCP 数据包并设置参数
使用 iptables 设置 mark 值（IPv4）
sudo iptables -t mangle -A POSTROUTING -p tcp -j MARK --set-mark 0xdeaf0000
位域含义：
0xdeaf0000 表示：
MARK_MAGIC = 0xdeaf（高16位）
其他位为默认值（如不修改序列号、无延迟等）。
自定义参数（按需调整）：
延迟时间：通过 MARK_DELAY（位 9-5）设置，例如 delay=5 → 0xdeaf0000 | (5 << 5)
重复次数：通过 MARK_REPEAT（位 12-10）设置，例如 repeat=3 → 0xdeaf0000 | (3 << 10)
破坏校验和：添加 MARK_WR_CHKSUM（位13）→ 0xdeaf0000 | (1 << 13)
更多组合：根据需求叠加位标志。
3. 动态修改发送缓冲区（DebugFS）
模块通过 /sys/kernel/debug/nf_deaf/buf 文件提供缓冲区配置：
查看当前缓冲区内容
cat /sys/kernel/debug/nf_deaf/buf
写入自定义内容（例如修改为 "USER testuser\r\n"）
echo "USER testuser" | sudo tee /sys/kernel/debug/nf_deaf/buf
限制：缓冲区最大长度为 256 字节（NF_DEAF_BUF_SIZE）。
4. 验证功能
场景：模拟 TCP 数据篡改和延迟
标记数据包：
sudo iptables -t mangle -A POSTROUTING -p tcp --dport 21 -j MARK --set-mark 0xdeafA020
0xdeafA020 解析：
MARK_MAGIC = 0xdeaf
MARK_DELAY = 0x14（十进制20 → 延迟20个jiffies，约200ms）
MARK_REPEAT = 0x2（重复2次）
MARK_WR_CHKSUM = 1（破坏校验和）
发送 FTP 请求：
curl ftp://example.com
抓包验证：
使用 tcpdump 或 Wireshark 观察：
原始包被篡改（序列号、校验和变化）。
数据包被重复发送。
延迟生效（时间戳间隔约200ms）。
5. 卸载模块
sudo rmmod nf_deaf
清理 iptables 规则：
sudo iptables -t mangle -F
