# R²-Gaussian 环境配置指南（Ubuntu 24.04 + CUDA 12）

> 适用于 Ubuntu 24.04 + NVIDIA GPU 环境，解决官方安装脚本在现代 Linux 发行版上的兼容性问题

## 问题背景

官方 README 提供的安装方式是为 Ubuntu 20.04 / CUDA 11.x 设计的。在 Ubuntu 24.04 上存在以下问题：

1. **CUDA 11.8 与 GCC 13 不兼容**：NVCC 报错 `unsupported GNU version! gcc versions later than 11 are not supported`
2. **系统头文件不兼容**：CUDA 11.8 编译时无法识别 `_Float32` 等类型
3. **CUDA 扩展缺少头文件**：`xray-gaussian-rasterization-voxelization` 源码缺少 `<cstdint>` 头文件

---

## 一、基础环境配置

### 1.1 创建 conda 环境

```bash
# 官方方式
conda env create --file environment.yml

# 或手动创建
conda create -n r2_gaussian python=3.9 -y
conda activate r2_gaussian
```

### 1.2 安装 PyTorch（关键差异）

| 官方方式 | 本文档方式 |
|---------|-----------|
| CUDA 11.8 | CUDA 12.1 |
| `torch==2.1.2+cu118` | `torch==2.1.2+cu121` |

```bash
# 卸载旧版本（如有）
pip uninstall -y torch torchvision

# 安装 CUDA 12.1 版本
pip install torch==2.1.2+cu121 torchvision==0.16.2+cu121 --extra-index-url https://download.pytorch.org/whl/cu121
```

**注意**：系统需要安装 CUDA 12.x

```bash
# 检查系统CUDA版本
ls /usr/local/ | grep cuda
# 输出: cuda cuda-12 cuda-12.8
```

### 1.3 安装其他依赖

```bash
pip install -r requirements.txt
```

---

## 二、编译 CUDA 扩展

### 2.1 simple-knn

**官方方式**：
```bash
pip install -e r2_gaussian/submodules/simple-knn
```

**本文档方式**：

由于 editable 模式在当前环境存在问题，采用手动编译安装：

```bash
cd r2_gaussian/submodules/simple-knn
export CUDA_HOME=/usr/local/cuda-12
python setup.py build
mkdir -p $CONDA_PREFIX/lib/python3.9/site-packages/simple_knn
cp simple_knn/_C.cpython-39-x86_64-linux-gnu.so $CONDA_PREFIX/lib/python3.9/site-packages/simple_knn/
touch $CONDA_PREFIX/lib/python3.9/site-packages/simple_knn/__init__.py
```

### 2.2 xray-gaussian-rasterization-voxelization

**官方方式**：
```bash
pip install -e r2_gaussian/submodules/xray-gaussian-rasterization-voxelization
```

**本文档方式**：

需要修复代码中缺少的头文件，然后手动编译：

#### 2.2.1 修复代码问题

**文件1**: `r2_gaussian/submodules/xray-gaussian-rasterization-voxelization/cuda_rasterizer/rasterizer_impl.h`

在 `#include <cuda_runtime_api.h>` 后添加：
```cpp
#include <cstdint>
```

**文件2**: `r2_gaussian/submodules/xray-gaussian-rasterization-voxelization/cuda_voxelizer/voxelizer_impl.h`

同样添加：
```cpp
#include <cstdint>
```

#### 2.2.2 编译并安装

```bash
cd r2_gaussian/submodules/xray-gaussian-rasterization-voxelization
export CUDA_HOME=/usr/local/cuda-12
python setup.py build

# 创建包目录
mkdir -p $CONDA_PREFIX/lib/python3.9/site-packages/xray_gaussian_rasterization_voxelization

# 复制Python文件
cp -r xray_gaussian_rasterization_voxelization/* $CONDA_PREFIX/lib/python3.9/site-packages/xray_gaussian_rasterization_voxelization/

# 复制编译好的.so文件
cp build/lib.linux-x86_64-cpython-39/xray_gaussian_rasterization_voxelization/_C.cpython-39-x86_64-linux-gnu.so $CONDA_PREFIX/lib/python3.9/site-packages/xray_gaussian_rasterization_voxelization/
```

---

## 三、配置环境变量

### 3.1 创建 conda 激活脚本（推荐）

```bash
mkdir -p $CONDA_PREFIX/etc/conda/activate.d/
cat > $CONDA_PREFIX/etc/conda/activate.d/cuda.sh << 'EOF'
export CUDA_HOME=/usr/local/cuda-12
export PATH=/usr/local/cuda-12/bin:$PATH
EOF
```

### 3.2 验证安装

```bash
conda activate r2_gaussian

python -c "
import torch
import simple_knn
import xray_gaussian_rasterization_voxelization

print('✓ All imports successful!')
print(f'  PyTorch: {torch.__version__}')
print(f'  CUDA available: {torch.cuda.is_available()}')
print(f'  CUDA version: {torch.version.cuda}')
"
```

预期输出：
```
✓ All imports successful!
  PyTorch: 2.1.2+cu121
  CUDA available: True
  CUDA version: 12.1
```

---

## 四、完整安装脚本

将以下内容保存为 `install.sh`：

```bash
#!/bin/bash
set -e

echo "=== R²-Gaussian Environment Setup ==="
echo ""

# 激活环境
conda activate r2_gaussian

# 安装 PyTorch with CUDA 12
echo "[1/5] Installing PyTorch with CUDA 12..."
pip uninstall -y torch torchvision 2>/dev/null || true
pip install torch==2.1.2+cu121 torchvision==0.16.2+cu121 --extra-index-url https://download.pytorch.org/whl/cu121

# 安装依赖
echo "[2/5] Installing dependencies..."
pip install -r requirements.txt

# 编译 simple-knn
echo "[3/5] Building simple-knn..."
cd r2_gaussian/submodules/simple-knn
export CUDA_HOME=/usr/local/cuda-12
python setup.py build
mkdir -p $CONDA_PREFIX/lib/python3.9/site-packages/simple_knn
cp simple_knn/_C.cpython-39-x86_64-linux-gnu.so $CONDA_PREFIX/lib/python3.9/site-packages/simple_knn/
touch $CONDA_PREFIX/lib/python3.9/site-packages/simple_knn/__init__.py

# 编译 xray-gaussian-rasterization-voxelization
echo "[4/5] Building xray-gaussian-rasterization-voxelization..."
cd ../../xray-gaussian-rasterization-voxelization
python setup.py build
mkdir -p $CONDA_PREFIX/lib/python3.9/site-packages/xray_gaussian_rasterization_voxelization
cp -r xray_gaussian_rasterization_voxelization/* $CONDA_PREFIX/lib/python3.9/site-packages/xray_gaussian_rasterization_voxelization/
cp build/lib.linux-x86_64-cpython-39/xray_gaussian_rasterization_voxelization/_C.cpython-39-x86_64-linux-gnu.so $CONDA_PREFIX/lib/python3.9/site-packages/xray_gaussian_rasterization_voxelization/

# 配置环境变量
echo "[5/5] Setting up environment variables..."
mkdir -p $CONDA_PREFIX/etc/conda/activate.d/
cat > $CONDA_PREFIX/etc/conda/activate.d/cuda.sh << 'EOF'
export CUDA_HOME=/usr/local/cuda-12
export PATH=/usr/local/cuda-12/bin:$PATH
EOF

echo ""
echo "=== Setup Complete ==="
echo "Please run: conda activate r2_gaussian"
```

---

## 五、可能遇到的问题

### Q1: 编译时报错 `unsupported GNU version`

**原因**：系统 GCC 版本过高（>=12），CUDA 11.8 不支持  
**解决**：使用 CUDA 12.x + PyTorch cu121

### Q2: 编译时报错 `identifier "_Float32" is undefined`

**原因**：CUDA 11.8 与 Ubuntu 24.04 系统头文件不兼容  
**解决**：同上，使用 CUDA 12.x

### Q3: editable 模式安装失败

**原因**：torch.utils.cpp_extension 在当前环境下的兼容性问题  
**解决**：手动编译并复制 .so 文件到 site-packages

### Q4: 运行时找不到 simple_knn 模块

**原因**：editable 安装的路径映射问题  
**解决**：按照 2.1 节手动复制 .so 文件

---

## 六、与官方安装的差异总结

| 步骤 | 官方方式 | 本文方式 |
|------|---------|---------|
| PyTorch | cu118 | cu121 |
| simple-knn | `pip install -e .` | 手动编译+复制 |
| xray-gaussian-rasterization | `pip install -e .` | 修复头文件+手动编译+复制 |
| CUDA_HOME | 未设置 | conda 激活脚本自动设置 |
| 系统要求 | Ubuntu 20.04 | Ubuntu 24.04 |

---

## 七、参考

- 官方 GitHub: https://github.com/Ruyi-Zha/r2_gaussian
- Arxiv 论文: https://arxiv.org/abs/2405.20693
