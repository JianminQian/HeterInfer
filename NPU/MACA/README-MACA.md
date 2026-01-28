
#  LMDeploy on MACA —— Decode 实例部署指南

> 本文档描述如何在 **MACA 设备节点** 上部署 LMDeploy 的 **Decode 实例**，用于异构推理中的解码阶段。需配合 PPU 上的 Prefill 实例与 Proxy 使用。

---

##  前提条件

- 已有 Qwen3-32B 模型（路径示例：`/datapool/tangzhiyi/data/Qwen3-32B`）
- 可访问阿里云容器镜像仓库

---

## 1️ 拉取 MACA 镜像

```bash
docker pull crpi-4crprmm5baj1v8iv.cn-hangzhou.personal.cr.aliyuncs.com/lmdeploy_dlinfer/maca:latest
```

---

## 2️ 启动 MACA 容器

```bash
docker run -itd \
   --privileged \
   --ipc host \
   --cap-add SYS_PTRACE \
   --device=/dev/mem \
   --device=/dev/dri \
   --device=/dev/mxcd \
   --device=/dev/infiniband \
   --group-add video \
   --network=host \
   --shm-size 400g \
   --ulimit memlock=-1 \
   --security-opt seccomp=unconfined \
   --security-opt apparmor=unconfined \
   -h "$(hostname)" \
   --name lt_maca \
   -v /mnt:/mnt \
   -v /datapool:/datapool \
   -v /var/run/docker.sock:/var/run/docker.sock \
   -w /datapool/lt \
   --entrypoint /bin/bash \
   crpi-4crprmm5baj1v8iv.cn-hangzhou.personal.cr.aliyuncs.com/lmdeploy_dlinfer/maca:latest
```

---

## 3️3 容器内环境配置

进入容器：

```bash
docker exec -it lt_maca bash
```

创建并运行初始化脚本：

```bash
cat > setup.sh << 'EOF'
#!/bin/bash
unset PIP_INDEX_URL
cd /datapool/tangzhiyi/hetero_maca/dev/lmdeploy && \
LMDEPLOY_TARGET_DEVICE=maca pip3 install -e . && \
cd /datapool/tangzhiyi/hetero_maca/dev/dlinfer && \
pip3 install -r requirements/maca/full.txt && \
DEVICE=maca python3 setup.py develop && \
pip install dlslime==0.0.1.post10
EOF

chmod +x setup.sh && ./setup.sh
```


## 4️4 启动 Decode 实例

####  前提条件
- 先启动Proxy 服务,（IP: `10.201.6.24`，端口: `8000`）

---

确保 RDMA 网络可用（如 `mlx5_bond_0`）：

```bash
export SLIME_VISIBLE_DEVICES=mlx5_bond_0
```

启动服务（注意 `--server-name` 为本机 IP）：

```bash
lmdeploy serve api_server \
  /datapool/tangzhiyi/data/Qwen3-32B \
  --model-name pd_test \
  --server-name 10.201.6.10 \
  --server-port 23333 \
  --role Decode \
  --proxy-url http://10.201.6.24:8000 \
  --backend pytorch \
  --device maca \
  --cache-block-seq-len 16 \
  --tp 4
```

>  `--proxy-url` 必须指向已运行的 Proxy 地址。

