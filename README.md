# MCS Editor - Minecraft 服务器列表编辑器

<p align="center">
  <img src="https://img.shields.io/badge/Vue.js-3-4FC08D?style=for-the-badge&logo=vue.js" alt="Vue 3" />
  <img src="https://img.shields.io/badge/Vite-5-646CFF?style=for-the-badge&logo=vite" alt="Vite" />
  <img src="https://img.shields.io/badge/TypeScript-blue?style=for-the-badge&logo=typescript" alt="TypeScript" />
  <img src="https://img.shields.io/badge/license-UNLICENSED-lightgrey?style=for-the-badge" alt="License" />
</p>

> [!NOTE]
> **关于代码质量：** 本项目代码主要由 Google Gemini 模型生成。可读性与可维护性或许不佳，请潜在的贡献者知悉。

<p align="center">
  一个为 Minecraft 服务器列表提供可视化配置的 Web 编辑器。
  <br />
  用户可以通过拖拽、表单等方式轻松构建复杂的服务器层级关系，并导出为特定格式的数据。
</p>

---

## ✨ 功能亮点

- **可视化操作**：通过直观的图形界面添加、编辑、删除服务器。
- **拖拽排序**：使用拖拽轻松调整服务器的显示优先级。
- **层级关系**：支持将服务器拖拽到另一个服务器上，以创建“父-子”服务器层级。
- **丰富的自定义选项**：为每个服务器设置独立的 IP、注释、标签和自定义颜色。
- **快捷预设**：内置多种常用服务器类型（如大厅、生存、PVP），一键填充。
- **导入/导出**：
  - 支持从 URL、JSON 或专用压缩字符串导入配置。
  - 可导出为 JSON 文件或生成用于 QQ 机器人的 `/mcs import` 命令。
- **明暗主题切换**：支持浅色与深色两种主题，适应不同环境下的使用习惯。

## 🖼️ 界面预览



<p align="center">
  <img src="https://raw.githubusercontent.com/FlyingPig278/mcs-editor/master/.github/screenshot.png" alt="MCS Editor 界面截图" width="80%" />
</p>

## 🚀 如何使用

1.  **访问编辑器**：打开部署好的网页地址。
2.  **添加服务器**：点击右上角的 `+ 添加服务器` 按钮，在弹出的窗口中填写服务器信息。
3.  **调整列表**：
    - 按住服务器卡片左侧的拖拽柄 `⠿` 上下拖动，以调整其在列表中的顺序。
    - 将一个服务器卡片拖拽到另一个卡片上，即可将其设置为子服务器。
4.  **导入配置**：在右侧的“导入”面板中，粘贴已有的分享链接或数据字符串，点击 `加载配置`。
5.  **导出配置**：配置完成后，在右侧的“导出”面板中，点击 `复制导入命令` 或 `导出JSON` 获取结果。

## 🛠️ 本地开发

1.  **克隆仓库**

    ```bash
    git clone https://github.com/FlyingPig278/mcs-editor.git
    cd mcs-editor
    ```

2.  **安装依赖**

    ```bash
    npm install
    ```

3.  **启动开发服务器**

    ```bash
    npm run dev
    ```

    随后在浏览器中打开对应的本地地址即可。

## 💻 技术栈

- **前端框架**: [Vue 3](https://vuejs.org/)
- **构建工具**: [Vite](https://vitejs.dev/)
- **编程语言**: [TypeScript](https://www.typescriptlang.org/)
- **拖拽库**: [vuedraggable](https://github.com/SortableJS/vuedraggable)
- **UI 风格**: 自定义 CSS，包含响应式设计和主题切换
- **数据压缩**: [pako](https://github.com/nodeca/pako) (用于生成紧凑的分享字符串)

---

*This project was scaffolded and coded by Google's Gemini.*