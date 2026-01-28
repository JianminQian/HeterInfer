# LMDeploy 异构推理部署总览

本项目支持在 **PPU** 和 **MACA** 两种异构硬件上协同部署 LMDeploy 的 Prefill/Decode 实例，通过 Proxy 统一调度。

## 📚 平台专属部署指南

- [PPU 节点部署（Prefill 实例）](./PPU/README-PPU.md)
- [MACA 节点部署（Decode 实例）](./MACA/README-MACA.md)

> 💡 Proxy 服务可以部署在 PPU/MACA 节点上（如部署在PPU上的IP: `10.201.6.24`），具体启动方式见各平台文档。


###  任意ppu/maca 环境上，启动Proxy
启动服务（注意 `--server-name` 为本机 ）：
```
lmdeploy serve proxy \
  --server-name 10.201.6.24 \
  --server-port 8000 \
  --routing-strategy "min_expected_latency" \
  --serving-strategy DistServe \
  --log-level INFO
```

>  `--server-port` 必须指向已运行的 Proxy 端口。