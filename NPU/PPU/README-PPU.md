#  LMDeploy on PPU —— Prefill 实例部署指南

> 本文档描述如何在 **PPU 设备节点** 上部署 LMDeploy 的 **Prefill 实例**，用于异构推理中的预填充阶段。需配合 MACA 上的 Decode 实例与 Proxy 使用。

---

## 前提条件

- 已有 Qwen3-32B 模型（路径：`/mnt/datapool/tangzhiyi/data/Qwen3-32B`）
- 已获取 PPU 推理镜像（`v1.5.2-pytorch2.6.0-ubuntu24.04-cuda12.6-vllm0.8.5-py312:v1`）


---

## 1 准备 PPU 镜像

若本地无镜像，从共享位置加载：

```bash
docker load -i /datapool/lt/llm_ppu_image.tar
```

---

## 2️ 启动 PPU 容器

```bash
docker run --privileged=true \
  --name lt_hetero \
  --device=/dev/alixpu_ppu0 \
  --device=/dev/alixpu_ppu1 \
  --device=/dev/alixpu \
  --device=/dev/alixpu_ctl \
  --ipc=host \
  --network=host \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  --init -td \
  --shm-size=500g \
  -v /mnt:/mnt \
  -v /datapool:/datapool \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -w /datapool/tangzhiyi/hetero_ppu \
  v1.5.2-pytorch2.6.0-ubuntu24.04-cuda12.6-vllm0.8.5-py312:v1
```

---

## 3️ 容器内环境配置

进入容器：

```bash
docker exec -it lt_hetero bash
```

### 方式一：使用 PYTHONPATH（避免重复安装）

```bash
unset PIP_INDEX_URL
export PYTHONPATH="/datapool/tangzhiyi/hetero_ppu/dev/lmdeploy:$PYTHONPATH"
```

> 可将此行加入 `~/.bashrc` 持久化。

### 方式二：重新安装 lmdeploy

```bash
unset PIP_INDEX_URL
cd /datapool/tangzhiyi/hetero_ppu/dev/lmdeploy
pip3 install -e .
```

---

## 4️ 启动 Prefill 实例

####  前提条件
- 先启动Proxy 服务,（IP: `10.201.6.24`，端口: `8000`）

---

启用 RDMA 网络：

```bash
export SLIME_VISIBLE_DEVICES=mlx5_bond_0
```

启动服务：

```bash
lmdeploy serve api_server \
  /mnt/datapool/tangzhiyi/data/Qwen3-32B \
  --model-name pd_test \
  --server-name 10.201.6.24 \
  --server-port 23333 \
  --role Prefill \
  --proxy-url http://10.201.6.24:8000 \
  --backend pytorch \
  --device cuda \
  --cache-block-seq-len 16 \
  --tp 4
```

> 🔔 `--server-name` 应为本机 IP。



