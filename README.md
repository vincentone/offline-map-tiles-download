# 地图瓦片下载器

 Vue.js + OpenLayers 构建的 Web 应用程序，旨在允许用户选择中国的行政区域，在地图上绘制矩形区域，并将指定区域内的地图瓦片下载为 ZIP 文件。并支持使用 GeoJSON 数据加载行政边界。

## 功能
- **行政区域预览**：可选择中国省/市，加载行政边界 GeoJSON 数据,方便预览范围。(数据来自阿里云数据可视化平台（`https://datav.aliyun.com/portal/school/atlas/area_selector`）)
- **绘制矩形**：在地图上绘制矩形区域以定义下载区域。
- **输入坐标**：可手动输入坐标以定义下载区域。
- **极速下载**：将选定区域内的地图瓦片按指定缩放级别下载为 ZIP 文件。
- **进度跟踪**：显示下载进度条和预计下载时间。
- **缩放级别信息**：在下载前通过表格查看缩放级别、瓦片数量和预计下载时间。

## 前提条件
- Node.js（v16 或更高版本）
- npm 或 yarn
- 现代浏览器（Chrome、Firefox 等）

## 安装
1. 克隆仓库：
   ```bash
   git clone <repository-url>
   cd map-tile-downloader
   ```
2. 安装依赖：
   ```bash
   npm install
   ```
   或
   ```bash
   yarn install
   ```
3. 启动开发服务器：
   ```bash
   npm run dev
   ```
   或
   ```bash
   yarn dev
   ```

## 依赖
- **Vue.js**：用于构建应用程序的前端框架。
- **OpenLayers**：用于渲染交互式地图的库。
- **Element Plus**：Vue.js 的 UI 组件库。
- **JSZip**：用于创建 ZIP 文件的库。
- **file-saver-fixed**：用于在客户端保存文件的库。

## 注意事项
- 默认地图来源高德，地图瓦片从 `http://wprd04.is.autonavi.com` 获取。可手动替换为自己所需要的地图来源。
- 测试时可取消代码中测试 URL 的注释`http://localhost:5173/tiles/{z}/{x}/{y}.png`, 并将下载的tiles.zip解压到public目录下。
- 默认的下载目录结构为tiles/y/x/y.png。


## 项目结构
```
├── public/                  # 静态资源
├── src/
│   ├── components/          # Vue 组件
│   ├── App.vue              # 主 Vue 组件（包含提供的代码）
│   ├── assets/              # CSS 和其他资源
│   └── main.js              # Vue 应用程序入口
├── package.json             # 项目依赖和脚本
└── README.md                # 本文件
```

