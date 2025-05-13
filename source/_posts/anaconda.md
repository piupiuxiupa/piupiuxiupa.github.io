---
title: Anaconda Install & Use
date: 2025-03-27 20:33:41
tags: 
  - tools
  - linux
  - python
categories: Posts
---

# Anaconda 安装

## Install

```bash
wget https://repo.anaconda.com/archive/Anaconda3-2023.07-2-Linux-x86_64.sh
bash Anaconda3-2023.07-2-Linux-x86_64.sh
source ~/.bashrc
# 配置国内源
cat >> ~/.condarc <<EOF
channels:
  - conda-forge
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
  - defaults
show_channel_urls: true
EOF

conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
conda config --add channels conda-forge

conda config --set show_channel_urls yes
```

## 常用命令

```bash
conda --version
conda init
conda activate base
conda config --show
conda config --show channels
conda clean -a
conda update conda
conda deactivate
conda create -n py310 python=3.10
conda list 
conda info
conda env list
conda env export -n myenv > myenv.yml
conda env create -f myenv.yml
conda list --export > conda_requirements.txt
conda install --file conda_requirements.txt
conda config --add channels conda-forge
```

## others

### gcc 源码安装

```bash
wget http://ftp.gnu.org/gnu/gcc/gcc-11.2.0/gcc-11.2.0.tar.gz
tar -zxvf gcc-11.2.0.tar.gz
cd gcc-11.2.0
# 下载必要的依赖
./contrib/download_prerequisites
mkdir build
cd build
../configure --enable-languages=c,c++ --disable-multilib --prefix=/usr/local/gcc-13.2.0
# 使用所有 CPU 核心来加速编译
make -j$(nproc) 
make install
# 配置环境变量，根据实际情况选择
export PATH=/usr/local/gcc-13.2.0/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/gcc-13.2.0/lib64:$LD_LIBRARY_PATH
source ~/.bashrc
```

### python封装成so文件

#### 安装cython

```bash
pip install cython
```

#### 创建编写源码文件及setup

```python
# example.py
def add(a, b):
    return a + b

def multiply(a, b):
    return a * b

# setup.py
from setuptools import setup
from Cython.Build import cythonize

setup(
    ext_modules=cythonize("example.py")
)
```

#### 运行setup

```bash
python setup.py build_ext --inplace
```

编译成功后会有一个类似`example.cpython-310-x86_64-linux-gnu.so`的文件

- .so 文件必须在当前工作目录下，或者在 PYTHONPATH 中。
- 如果你想放在其他目录下，可以手动修改 sys.path：

```python
import sys
sys.path.append('/path/to/your/so/file')

import example
print(example.add(5, 3))  # 8
```

#### 在python中导入so文件

```python
import example

print(example.add(5, 3))      # 输出: 8
print(example.multiply(5, 3)) # 输出: 15
```
