---
layout: post
title: "Ubuntu 下 conda 设置"
date:  2025-03-23 09:50:07 +0800
categories: ["技术", "编程", "Linux和虚拟机"]
tag: ["Ubuntu", "conda"]
---

# 禁止自动进入 base 环境

Ubuntu 安装 Conda/MiniConda 之后，打开终端默认自动激活 base 环境。

一般我们也不使用 base 环境，考虑到每次都通过 `conda deactivate` 退出也比较麻烦,可以通过修改 conda 的 config 文件来禁止这一行为：

```
conda config --set auto_activate_base false
```

# 增加 Tab 补全

1. 安装 `conda-bash-completion` 插件

```
conda install -c conda-forge conda-bash-completion
```

2. 禁止自动激活 base 环境下设置
在设置 `auto_activate_base: false` 的时候，还需要再 `~/.bashrc` 中添加下列内容，自动补全才会生效：

```
CONDA_ROOT=~/miniconda3  # <set to your Anaconda/Miniconda installation directory>
if [[ -r $CONDA_ROOT/etc/profile.d/bash_completion.sh ]]; then
    source $CONDA_ROOT/etc/profile.d/bash_completion.sh
else
    echo "WARNING: counld not find conda-bash-completion setup script"
fi
```

关闭 Terminal 并重新打开，或者：

```
source ~/.bashrc
```