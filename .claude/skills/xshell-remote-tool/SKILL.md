---
name: xshell-remote-tool
description: Electron 桌面远程连接工具 — SSH 终端、串口调试、SFTP 文件传输。React + TypeScript + xterm.js + ssh2 + serialport。
---

# XShell Remote Tool

基于 Electron 的远程连接桌面应用，支持 SSH 终端、串口调试、SFTP 文件管理，类似 XShell 的功能。

## 项目结构

```
xshell-clone/
├── src/
│   ├── main/                   # Electron 主进程
│   │   ├── index.ts            # 入口：创建窗口、注册 IPC、生命周期
│   │   ├── ipc/
│   │   │   ├── index.ts        # registerAllHandlers()
│   │   │   ├── ssh.ipc.ts      # ssh:connect / disconnect / data / resize
│   │   │   ├── serial.ipc.ts   # serial:connect / disconnect / listPorts / data
│   │   │   ├── sftp.ipc.ts     # sftp:open / close / listDir / upload / download / mkdir / delete / rename
│   │   │   ├── session.ipc.ts  # session:list / get / save / delete（SQLite 持久化）
│   │   │   └── fs.ipc.ts       # 本地文件系统操作
│   │   ├── ssh/
│   │   │   ├── ssh-manager.ts  # SSHManager: 管理多 SSH 会话（Map<id, SSHSession>）
│   │   │   ├── ssh-session.ts  # SSHSession: 单连接生命周期 + PTY
│   │   │   ├── sftp-manager.ts # SFTPManager: 管理多 SFTP 会话
│   │   │   └── sftp-session.ts # SFTPSession: 文件列表/上传/下载
│   │   ├── serial/
│   │   │   ├── serial-manager.ts
│   │   │   └── serial-session.ts
│   │   └── store/
│   │       ├── database.ts     # sql.js 初始化 + sessions / settings 表
│   │       └── session-store.ts# CRUD：StoredSession (SQLite)
│   ├── preload/
│   │   └── index.ts            # contextBridge 暴露 API 给渲染进程
│   └── renderer/               # React 渲染进程
│       ├── index.tsx / App.tsx # 入口组件：布局 + 快捷键 Ctrl+O / Ctrl+B
│       ├── components/
│       │   ├── layout/
│       │   │   ├── AppShell.tsx  # 整体布局：侧边栏 + 主内容区
│       │   │   └── StatusBar.tsx # 底部状态栏（连接状态/行列数）
│       │   ├── terminal/
│       │   │   ├── TabBar.tsx    # 标签页管理
│       │   │   └── TerminalView.tsx # xterm.js 终端渲染
│       │   ├── connections/
│       │   │   ├── QuickConnectDialog.tsx # 快速连接弹窗（SSH/串口）
│       │   │   └── SessionTree.tsx        # 左侧会话树
│       │   └── sftp/
│       │       ├── SFTPPanel.tsx    # SFTP 面板（文件列表 + 操作）
│       │       ├── FileList.tsx     # 文件列表组件
│       │       └── TransferQueue.tsx# 传输队列
│       ├── store/
│       │   ├── connection-store.ts # Zustand: tabs[], activeTabId, status
│       │   └── ui-store.ts         # Zustand: sidebarOpen, dialogs
│       ├── hooks/
│       │   ├── useTerminal.ts      # xterm.js Terminal 实例管理
│       │   └── useSerialPorts.ts   # 串口列表轮询
│       ├── styles/global.css
│       └── types/global.d.ts
├── electron.vite.config.ts  # electron-vite 构建配置
├── electron-builder.yml     # electron-builder 打包配置（NSIS 安装包）
├── tsconfig.json            # TypeScript 项目引用
├── tsconfig.node.json       # 主进程 TS 配置
└── tsconfig.web.json        # 渲染进程 TS 配置
```

## 技术栈

| 层 | 技术 |
|---|------|
| 桌面框架 | Electron 33 |
| 构建工具 | electron-vite 2.x + Vite 5.x |
| 前端 | React 19 + TypeScript 5.6 |
| UI 组件库 | @fluentui/react-components 9.x |
| 终端 | @xterm/xterm 5.x + addon-fit + addon-search + addon-web-links |
| 状态管理 | Zustand 5 |
| SSH | ssh2 1.x |
| 串口 | serialport 12.x |
| 数据库 | sql.js 1.x（SQLite WASM） |
| 打包 | electron-builder 25.x → NSIS 安装包 |

## 架构

```
渲染进程 (React)         主进程 (Electron)
┌─────────────────┐     ┌──────────────────────┐
│  App.tsx         │     │  index.ts            │
│  ├─ AppShell     │     │  ├─ registerAllHandlers()
│  ├─ TabBar       │     │  ├─ SSHManager        │
│  ├─ TerminalView │◄───►│  ├─ SerialManager     │
│  ├─ SFTPPanel    │ IPC │  ├─ SFTPManager       │
│  ├─ SessionTree  │     │  ├─ SessionStore      │
│  └─ StatusBar    │     │  └─ Database (sql.js) │
└─────────────────┘     └──────────────────────┘
```

- **通信方式**：`contextBridge` + `ipcMain.handle` / `ipcRenderer.invoke`
- **终端数据**：SSH/串口 → Node.js Stream → webContents.send → xterm.js write
- **用户输入**：xterm.onData → ipcRenderer.send → ssh2/SerialPort.write
- **持久化**：sql.js WASM 模式（数据库文件在 `app.getPath('userData')/xshell-data.db`）

## 常用命令

```bash
# 开发模式（热重载）
npm run dev

# 构建
npm run build

# 打包为 Windows 安装包 (dist/)
npm run package
```

## 开发指南

### 添加新的 IPC 通道
1. 在 `src/main/ipc/` 对应的 .ts 文件中添加 `ipcMain.handle`
2. 在 `src/preload/index.ts` 中通过 `contextBridge.exposeInMainWorld` 暴露给渲染进程
3. 在渲染进程通过 `window.api.xxx()` 调用

### 添加新的 React 组件
1. 在 `src/renderer/components/` 对应目录创建组件文件
2. 如需跨组件状态，在 `src/renderer/store/` 用 Zustand 扩展 store
3. 全局样式在 `src/renderer/styles/global.css`
4. UI 组件优先使用 `@fluentui/react-components`

### 数据库 schema 变更
- 表定义在 `src/main/store/database.ts` 的 `getDatabase()` 中
- CRUD 操作在 `src/main/store/session-store.ts`
- 数据库是 SQLite (sql.js)，ALTER TABLE 能力有限，注意向后兼容

### SSH 会话生命周期
```
SSHManager.connect(config) → new SSHSession(id, window) → session.connect()
  → ssh2.Client.connect() → on('ready') → shell() → stream
  → stream.on('data') → window.webContents.send('ssh:data', id, data)
  → 渲染进程 TerminalView → xterm.write(data)
```

### 串口会话生命周期
```
SerialManager.connect(config) → new SerialPort() → on('data')
  → window.webContents.send('serial:data', id, data)
  → 渲染进程 TerminalView → xterm.write(data)
```

### SFTP 依赖 SSH 连接
- SFTP 复用已有 SSH 连接的 ssh2 Client 实例
- `sftp:open` 需要传入 `sshSessionId`，通过 `sshManager.getClient()` 获取底层连接
- 上传/下载是 fire-and-forget 模式，进度通过独立的 transfer 事件推送

## 注意事项
- native 模块 (`ssh2`, `serialport`, `sql.js`) 必须在 `electron-vite` 配置中 externalize
- `contextIsolation: true` + `nodeIntegration: false` — 渲染进程不能直接用 Node API
- sql.js 是纯 WASM 实现，不需要本地编译，但大数据库会有性能瓶颈
- 打包时 `sql.js/dist` 需要作为 `extraResources` 包含进去
- npm scripts 使用 `electron-vite` 而非原生 `vite`
