以immortalwrt-mt798x 为例：
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
3. 编译完整固件
    make -j$(nproc) V=s
4. 仅编译 nf_deaf 模块
   make package/custom/nf_deaf/compile V=s
   编译结果位于bin/targets/<架构>/packages/kmod-nf-deaf_*.ipk
