# setup-guides

Ubuntu + ROS2 環境のセットアップ手順をまとめたリポジトリです。
自律移動ロボット開発（GLIM SLAM・ROS2 bag操作等）を中心に、環境構築でつまずいたポイントを記録しています。

## ガイド一覧

| ファイル | 内容 |
|---|---|
| [GLIM-setup.md](./GLIM-setup.md) | GLIM（LiDAR SLAM）のGPUビルド手順（CUDA 12.x / ROS2 Humble） |
| [gpu-setup.md](./gpu-setup.md) | Docker + NVIDIA GPU 環境構築 |
| [bag_topic_rename.md](./bag_topic_rename.md) | ROS2 bagファイルのトピック名変更（sqlite3 直接操作） |
| [japanese-input.md](./japanese-input.md) | Ubuntu 上の日本語入力設定 |

## 動作確認環境

| 項目 | バージョン |
|---|---|
| OS | Ubuntu 22.04 (Jammy) |
| ROS2 | Humble |
| Docker base image | `nvidia/cuda:12.2.0-cudnn8-devel-ubuntu22.04` |
| GPU | NVIDIA GeForce GTX 1650 |
| ホストドライバ対応CUDA | 12.2 |

## 注意

- GLIM を CUDA=ON でビルドするには **CUDA 12.x** が必須です（CUDA 11.x では `thrust::cuda::par_nosync` エラーが発生します）
- bag トピック名変更は `.db3` と `metadata.yaml` の**両方**を修正する必要があります

## 関連リポジトリ

- [KBKN-Autonomous-Robotics-Lab/orange2025](https://github.com/KBKN-Autonomous-Robotics-Lab/orange2025) — 自律走行ロボット本体コード
