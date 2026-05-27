# OpenClash_Overwrite 覆写模块

> 欢迎通过 [Issue](https://github.com/wangqiankun/OpenClash_Overwrite/issues) 反馈问题。  
> 适配版本：**OpenClash v0.47.006 及以上**  
> 当前仓库只保留 **Smart 主路由配置**。

---

<p align="center">
  <img src="https://img.shields.io/github/last-commit/wangqiankun/OpenClash_Overwrite?style=for-the-badge&logo=git&label=Last%20Update" />
  <img src="https://img.shields.io/github/repo-size/wangqiankun/OpenClash_Overwrite?style=for-the-badge&logo=github&label=Repo%20Size" />
  <a href="https://github.com/vernesong/OpenClash" target="_blank">
    <img src="https://img.shields.io/badge/OpenClash-v0.47.006%2B-blue?style=for-the-badge&logo=openwrt" />
  </a>
</p>

---

## 目录

* [使用建议](#使用建议)
* [Smart 配置说明](#smart-配置说明)
* [使用方法](#使用方法)
* [保留文件](#保留文件)
* [温馨提示](#温馨提示)

---

## 使用建议

* 建议切换更新分支为 **Dev** 并 **启用 Smart 内核**。  
* 分流规则与策略组基于以下项目：  
  👉 [Aethersailor/Custom_OpenClash_Rules](https://github.com/Aethersailor/Custom_OpenClash_Rules.git)  

---

## Smart 配置说明

* **核心类型**：`Smart`
* **LGBM 模型**：关闭，使用默认模型
* **数据收集**：开启，收集大小 `500 MB`
* **权重加成**：  
  * `Premium: 0.9`  
  * `SG: 1.3`  
  * `HK: 1.5`  

---

## 使用方法

### 新增覆写模块

* **文件名**：可自定义  
* **类型**：`http`  
* **订阅链接**：

```bash
https://gh-proxy.19930405.xyz/https://raw.githubusercontent.com/wangqiankun/OpenClash_Overwrite/refs/heads/main/Overwrite/Overwrite-smart.conf
```

### 配置环境变量

在 OpenClash 中启用覆写模块前，请根据需要设置以下环境变量：

#### 必填变量：机场订阅链接

仅支持单一订阅，多机场用户建议搭配 Sub-store 聚合后载入配置。

```bash
EN_KEY=你的机场订阅链接
```

配置完成后，**保存 → 应用配置重启即可**。  

---

## 保留文件

当前仓库只保留以下运行相关文件：

```text
Overwrite/Overwrite-smart.conf
Yaml/Overwrite-Clash.yaml
README.md
```

其中 `Overwrite/Overwrite-smart.conf` 会通过 `DOWNLOAD_FILE` 下载 `Yaml/Overwrite-Clash.yaml` 到 OpenClash 配置目录。

---

## 温馨提示

1. 仓库文件均使用 **GitHub 原始链接**，请确保路由器可正常访问 GitHub。  
2. 默认 OpenClash 不包含 **Geo 数据库**：
   * 建议手动提前更新 Geo 数据库。
   * 当前 YAML 文件不依赖 Geo 数据库运行，但仍建议保持数据库更新。
3. 如未提前下载 GeoIP 数据库，初次运行可能提示 **内核错误**，多次重启后或手动下载数据库后即可恢复。  
