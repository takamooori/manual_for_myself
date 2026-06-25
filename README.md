# Manual for Myself

Ubuntu + ROS2 環境のセットアップ・運用手順をまとめた個人用リポジトリ。
自律移動ロボット開発で**自分がつまずいたポイント**を中心に記録しています。

---

## 📖 ガイド一覧

### 🛰 GLIM（LiDAR-Inertial SLAM）

| ファイル | 内容 |
|---|---|
| [glim_build.md](./glim_build.md) | GLIM の GPU ビルド手順（CUDA 12.x / ROS2 Humble） |
| [glim_dump.md](./glim_dump.md) | GLIM dump ファイルの構造と使い方 |
| [glim_parameters.md](./glim_parameters.md) | 主要パラメータと dump 頻度・軌跡精度への影響 |

### 🤖 ROS2

| ファイル | 内容 |
|---|---|
| [ros2_bag_rename.md](./ros2_bag_rename.md) | ROS2 bag ファイルのトピック名変更（sqlite3 直接操作） |

### 🐳 Docker

| ファイル | 内容 |
|---|---|
| [docker_nvidia.md](./docker_nvidia.md) | Docker + NVIDIA GPU 環境の構築 |

### 💻 Ubuntu

| ファイル | 内容 |
|---|---|
| [ubuntu_japanese_input.md](./ubuntu_japanese_input.md) | Ubuntu 上の日本語入力設定 |

---

## 🖥 動作確認環境

| 項目 | バージョン |
|---|---|
| OS | Ubuntu 22.04 (Jammy) |
| ROS2 | Humble |
| Docker base image | `nvidia/cuda:12.2.0-cudnn8-devel-ubuntu22.04` |
| GPU | NVIDIA GeForce GTX 1650 |
| ホストドライバ対応 CUDA | 12.2 |

---

## ⚠️ ハマりポイント早見

- GLIM を `CUDA=ON` でビルドするには **CUDA 12.x が必須**（11.x では `thrust::cuda::par_nosync` エラー）
- bag のトピック名変更は `.db3` と `metadata.yaml` の **両方** を修正する必要あり

---

## 🔗 関連リポジトリ

- [KBKN-Autonomous-Robotics-Lab/orange2025](https://github.com/KBKN-Autonomous-Robotics-Lab/orange2025) — 自律走行ロボット本体コード
