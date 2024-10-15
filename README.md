# OpenROAD Project Learning

## 目录

1. [项目简介](#项目简介)
2. [OpenROAD项目的部署](#OpenROAD项目的部署)
3. [OpenROAD简介](#OpenROAD简介)
4. [OpenROAD流程使用](#OpenROAD流程使用)
5. [参考资料](#参考资料)

## 项目简介

本项目是记录学习OpenROAD相关笔记，旨在介绍和演示OpenROAD和OpenSTA这两个开源EDA工具的使用和功能。

## OpenROAD项目的部署

### 1. 引言

OpenROAD（Foundations and Realization of Open, Accessible Design）项目于2018年启动，旨在解决硬件设计中的高成本、专业知识门槛以及不可预测性问题。由Qualcomm、Arm、多所大学和合作伙伴共同开发，项目由加州大学圣地亚哥分校（UC San Diego）牵头，专注于数字系统芯片（SoC）设计中的RTL到GDSII阶段的全自动开源工具链开发。

### 2. 环境准备与安装

#### 安装依赖

```shell
git clone --recursive https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts
cd OpenROAD-flow-scripts
sudo ./setup.sh
```

#### 本地构建OpenROAD

```shell
./build_openroad.sh --local
```

#### 验证安装

```shell
source ./env.sh
yosys -help
openroad -help
cd flow
make
```

## OpenROAD简介

OpenROAD是Foundations and Realization of Open, Accessible Design的缩写，集成了一系列EDA开发工具，实现了由RTL到GDS的开发。包括逻辑综合、底层布局、布局、时钟树综合、布线、寄生提取和时序分析等阶段。

## OpenROAD流程使用

### 1. 逻辑综合 Logic Synthesis
### 2. 布局设计和电源分配网络 Floorplan and early power delivery network
### 3. 布局和电源分配网络细化 Placement and PDN refinement
### 4. 时钟树综合 Clock Tree Synthesis
### 5. 全局布线 Global routing
### 6. 详细布线 Detailed Routing
### 7. 静态时序分析 Static Timing Analysis
### 8. 寄生参数提取 Parasitic Extraction

## 参考资料

1. [OpenROAD GitHub Repository](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)

