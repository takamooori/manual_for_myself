# GLIM GPU セットアップマニュアル

## 前提条件

| 項目 | バージョン |
|------|-----------|
| OS | Ubuntu 22.04 (Jammy) |
| Docker ベースイメージ | `nvidia/cuda:12.2.0-cudnn8-devel-ubuntu22.04` |
| ホストドライバ対応CUDA | 12.2 |
| GPU | NVIDIA GeForce GTX 1650 |
| ROS2 | Humble |

> ⚠️ CUDA=ON でビルドするには CUDA 12.x 環境が必須。CUDA 11.8 では `thrust::cuda::par_nosync` エラーが発生する。

---

## 1. 依存パッケージのインストール

```bash
sudo apt install -y \
  libomp-dev \
  libboost-all-dev \
  libmetis-dev \
  libfmt-dev \
  libspdlog-dev \
  libglm-dev \
  libglfw3-dev \
  libpng-dev \
  libjpeg-dev
```

---

## 2. Iridescence（可視化ライブラリ）のインストール

> `ros2_ws` の外（ホームディレクトリ直下推奨）で実行する

```bash
cd ~
git clone https://github.com/koide3/iridescence --recursive
mkdir iridescence/build && cd iridescence/build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install
```

---

## 3. GTSAM のインストール（v4.3a0）

```bash
cd ~
git clone https://github.com/borglab/gtsam
cd gtsam && git checkout 4.3a0
mkdir build && cd build
cmake .. \
  -DGTSAM_BUILD_EXAMPLES_ALWAYS=OFF \
  -DGTSAM_BUILD_TESTS=OFF \
  -DGTSAM_WITH_TBB=OFF \
  -DGTSAM_USE_SYSTEM_EIGEN=ON \
  -DGTSAM_BUILD_WITH_MARCH_NATIVE=OFF
make -j$(nproc)
sudo make install
```

---

## 4. gtsam_points のインストール

```bash
cd ~
git clone https://github.com/koide3/gtsam_points
mkdir gtsam_points/build && cd gtsam_points/build
cmake .. -DBUILD_WITH_CUDA=ON
make -j$(nproc)
sudo make install
```

---

## 5. 共有ライブラリをシステムに反映

```bash
sudo ldconfig
```

---

## 6. GLIM ROS2 パッケージの clone

> `glim` / `glim_ros2` は `ros2_ws/src` に clone する

```bash
cd ~/ros2_ws/src
git clone https://github.com/koide3/glim
git clone https://github.com/koide3/glim_ros2
```

---

## 7. ビルド

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
colcon build
source ~/.bashrc
```

---

## 備考

- iridescence / gtsam / gtsam_points は `sudo make install` でシステムにインストールされるため、clone 場所はどこでもよい（`ros2_ws` 内への clone は非推奨）
- CUDA=OFF でビルドした場合は GPU 高速化なしで動作可能（CPU モード）
- CUDA=ON でビルドに失敗する場合は Docker ベースイメージの CUDA バージョンを確認する
