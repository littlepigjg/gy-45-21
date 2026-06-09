# CSS 精灵图工具

一个基于 React + TypeScript + Vite 的 CSS 精灵图（Sprite Sheet）生成与管理工具，支持图标上传、项目管理、精灵图生成与拆分。

---

## 图标数据完整流转过程详解

本应用中，一张图标从用户上传到最终在图标库中渲染展示，需要经过 **9 个核心环节**。以下是每个环节的详细说明、技术选型原因及可能的失败点。

### 整体流转概览

```
用户拖拽/点击上传
       ↓
  文件格式验证 (MIME 类型白名单)
       ↓
  读取为 DataURL (FileReader)
       ↓
  解析图片尺寸 (HTMLImageElement)
       ↓
  生成唯一 ID (时间戳+随机数)
       ↓
  转换为 Blob 对象 (base64 解码)
       ↓
  存入 IndexedDB (二进制存储)
       ↓
  提取元数据保存到 localStorage (JSON 序列化)
       ↓
  图标库列表渲染展示 (从 IndexedDB 读取 + img 标签渲染)
```

---

## 环节一：用户拖拽或点击上传

### 代码位置

- 精灵图生成器拖拽上传：[Generator.tsx#L324-L350](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/pages/Generator.tsx#L324-L350)
- 精灵图生成器点击上传：[Generator.tsx#L343-L350](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/pages/Generator.tsx#L343-L350)
- 图标库点击上传：[Library.tsx#L572-L579](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/pages/Library.tsx#L572-L579)

### 技术实现

应用提供两种上传方式，均使用标准 HTML5 API：

1. **点击上传**：通过隐藏的 `<input type="file" multiple accept="image/*">` 元素，点击按钮时触发 `inputRef.current?.click()`。
2. **拖拽上传**（仅 Generator 页面）：通过 React 的 `onDragOver`、`onDragLeave`、`onDrop` 事件监听文件拖放，从 `e.dataTransfer.files` 获取文件列表。

### 技术选型原因

- **原生 HTML5 File API**：无第三方依赖，所有现代浏览器均支持，性能最优。
- **`accept="image/*"`**：在系统文件选择器层面过滤非图片文件，减少前端无效处理。
- **`multiple` 属性**：支持批量上传，提升用户体验。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| 用户选择了非图片文件 | `accept="image/*"` 在某些浏览器/操作系统中不严格生效 | 后续环节有 MIME 类型二次验证 |
| 拖拽时文件被浏览器导航打开 | 未调用 `e.preventDefault()` | 代码中已在 `onDrop` 和 `onDragOver` 中调用 |
| 超大文件导致浏览器卡顿 | 单文件过大 | FileReader 是异步的，但超大图（>50MB）仍可能阻塞主线程 |

---

## 环节二：文件格式验证

### 代码位置

- 验证函数：[utils/index.ts#L58-L60](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L58-L60)
- 调用位置：[utils/index.ts#L77-L82](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L77-L82)

### 技术实现

```typescript
function isImageFile(file: File): boolean {
  return /^image\/(png|jpe?g|gif|webp|svg\+xml)$/i.test(file.type);
}
```

通过正则表达式对 `File.type`（MIME 类型）进行白名单校验，仅允许：
- `image/png`
- `image/jpeg` / `image/jpg`
- `image/gif`
- `image/webp`
- `image/svg+xml`

### 技术选型原因

- **MIME 白名单而非黑名单**：安全性更高，避免恶意文件伪装扩展名。
- **基于 `file.type` 而非扩展名**：`file.type` 由浏览器根据文件内容嗅探得出，比单纯检查扩展名更可靠。
- **正则严格匹配**：`^image/...$` 确保不匹配类似 `image/png; charset=utf-8` 的异常类型。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| SVG 文件被拒绝 | 某些系统上报的 MIME 类型可能是 `image/svg` 而非 `image/svg+xml` | 可考虑放宽正则或增加扩展名二次校验 |
| WebP 旧版本浏览器不支持 | 浏览器无法解析 WebP 图片 | 后续 `getImageSize` 会报错并被 Promise 捕获 |
| 文件扩展名伪造但内容真实 | 例如将 `.txt` 改为 `.png` 但内容是文本 | `file.type` 仍会识别为 `text/plain` 而被过滤 |

---

## 环节三：读取为 DataURL

### 代码位置

- 核心函数：[utils/index.ts#L11-L18](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L11-L18)
- 调用位置：[utils/index.ts#L62-L75](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L62-L75)

### 技术实现

```typescript
export function fileToDataUrl(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}
```

使用 HTML5 `FileReader` API 将 `File` 对象异步读取为 Base64 编码的 DataURL 字符串，格式如：
```
data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...
```

### 技术选型原因

- **DataURL 通用性强**：可直接赋值给 `<img src>`、`<canvas>` 绘制、通过网络传输等，是后续尺寸解析和页面渲染的中间格式。
- **Promise 封装**：将回调式 API 转为 Promise，便于 `async/await` 链式调用和错误处理。
- **异步非阻塞**：`FileReader` 读取大文件时不会阻塞 UI 线程。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| 文件读取权限被拒 | 浏览器安全策略限制 | `reader.onerror` 触发 reject，Promise 链中断 |
| 文件已被外部删除 | 拖拽后但读取前文件被移动/删除 | 读取失败触发 onerror |
| 内存溢出 | 单文件极大（>100MB），Base64 膨胀约 33% | 浏览器可能抛出 OutOfMemory 错误 |
| DataURL 字符串过长 | 长字符串传递可能触发内存拷贝 | 后续环节转为 Blob 存储 |

---

## 环节四：解析图片尺寸

### 代码位置

- 核心函数：[utils/index.ts#L20-L29](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L20-L29)
- 调用位置：[utils/index.ts#L62-L75](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L62-L75)

### 技术实现

```typescript
export function getImageSize(
  dataUrl: string
): Promise<{ width: number; height: number }> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve({ width: img.width, height: img.height });
    img.onerror = reject;
    img.src = dataUrl;
  });
}
```

创建一个内存中的 `HTMLImageElement`（不挂载到 DOM），将 DataURL 赋给 `src`，在 `onload` 回调中读取解码后的像素尺寸。

### 技术选型原因

- **浏览器原生解码**：利用浏览器内置的图像解码器（libpng、libjpeg 等），无需引入额外解码库，支持所有目标格式。
- **精确像素尺寸**：`img.width` / `img.height` 返回的是图像的真实像素数，而非 CSS 渲染尺寸。
- **SVG 支持**：对于 SVG，浏览器会解析其 `viewBox` 或固有尺寸。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| 图片文件损坏 | 文件内容截断或格式错误 | `img.onerror` 触发 reject |
| 跨域图片被污染 | 非同源 DataURL（本应用不存在此问题） | CORS 限制导致无法读取尺寸 |
| SVG 无固有尺寸 | SVG 未设置 width/height 且 viewBox 缺失 | 浏览器可能返回 0×0 或异常值 |
| 超大图解码耗时 | 4K+ 分辨率图片解码慢 | 异步 Promise 不会阻塞，用户可能感知延迟 |

---

## 环节五：生成唯一 ID

### 代码位置

- 核心函数：[utils/index.ts#L3-L5](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L3-L5)

### 技术实现

```typescript
export function generateId(): string {
  return Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
}
```

ID 由两部分拼接并以 **Base36**（0-9, a-z）编码：
1. **时间戳部分**：`Date.now()` 当前毫秒时间戳，约占 8-9 个字符。
2. **随机部分**：`Math.random()` 生成的浮点数取小数点后 6 位 Base36，占 6 个字符。

典型输出示例：`lq8x2k0a3b7c`

### 技术选型原因

- **无需 UUID 库**：减少依赖体积，对于单用户本地应用，碰撞概率可忽略不计。
- **时间有序**：前缀为时间戳，ID 天然带有时间顺序，便于按创建时间排序。
- **Base36 编码紧凑**：比十进制短约 40%，比十六进制短约 15%。
- **足够的随机性**：6 位 Base36 提供 `36^6 ≈ 21.7 亿` 种组合，加上毫秒时间戳，单机环境下几乎不可能碰撞。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| 同一毫秒内极大量上传 | 毫秒级时间戳相同 + 随机数碰撞 | 单机单用户场景极难发生，如需支持可加计数器 |
| `Math.random()` 伪随机 | 理论上可预测 | 本地应用不涉及安全，不构成问题 |
| 系统时间回拨 | NTP 校时导致时间戳倒退 | ID 前缀可能重复，但随机部分仍可区分 |

---

## 环节六：转换为 Blob 对象

### 代码位置

- 核心函数：[utils/db.ts#L98-L115](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/db.ts#L98-L115)
- 调用位置：[utils/db.ts#L45-L48](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/db.ts#L45-L48)

### 技术实现

```typescript
export function dataUrlToBlob(dataUrl: string): Promise<Blob> {
  return new Promise((resolve, reject) => {
    try {
      const arr = dataUrl.split(',');
      const mimeMatch = arr[0].match(/:(.*?);/);
      const mime = mimeMatch ? mimeMatch[1] : 'image/png';
      const bstr = atob(arr[1]);
      let n = bstr.length;
      const u8arr = new Uint8Array(n);
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
      }
      resolve(new Blob([u8arr], { type: mime }));
    } catch (e) {
      reject(e);
    }
  });
}
```

转换步骤：
1. 以 `,` 分割 DataURL，分离头部（含 MIME）和 Base64 数据。
2. 正则提取 MIME 类型，默认为 `image/png`。
3. `atob()` 将 Base64 字符串解码为二进制字符串。
4. 将二进制字符串逐字符转为 `Uint8Array` 字节数组。
5. 用字节数组构造 `Blob` 对象并携带正确的 MIME 类型。

### 技术选型原因

- **Blob 更适合存储**：二进制数据比 Base64 字符串节省约 33% 存储空间（Base64 每 3 字节编码为 4 字符）。
- **IndexedDB 原生支持 Blob**：IndexedDB 可直接存储 Blob/File 对象，无需额外序列化。
- **手动解析而非 `fetch(dataUrl).then(r => r.blob())`**：避免创建微任务队列，同步解析更快，且不依赖网络栈。
- **保留 MIME 类型**：后续从 IndexedDB 读取转回 DataURL 时能正确还原类型。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| DataURL 格式非法 | 缺少 `,` 分隔符或 Base64 损坏 | `split` / `atob` 抛出异常，Promise reject |
| MIME 类型解析失败 | DataURL header 格式异常 | 回退默认值 `image/png` |
| 内存占用峰值 | `atob` 生成二进制字符串 + `Uint8Array` 两份拷贝 | 大文件时内存短暂翻倍，正常后 GC 回收 |
| `atob` 在极端环境不支持 | 极旧浏览器 | 现代浏览器均支持，Vite 构建目标已限定 |

---

## 环节七：存入 IndexedDB

### 代码位置

- 数据库初始化：[utils/db.ts#L13-L31](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/db.ts#L13-L31)
- 保存 Blob：[utils/db.ts#L33-L43](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/db.ts#L33-L43)
- 保存入口：[utils/db.ts#L45-L48](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/db.ts#L45-L48)
- Store 层调用：[useAppStore.ts#L114-L133](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/store/useAppStore.ts#L114-L133)

### 技术实现

```typescript
// 数据库配置
const DB_NAME = 'sprite-lab-db';
const DB_VERSION = 1;
const STORE_ICONS = 'icons';

// 打开数据库（单例 Promise 缓存）
let dbPromise: Promise<IDBDatabase> | null = null;

function openDB(): Promise<IDBDatabase> {
  if (dbPromise) return dbPromise;
  dbPromise = new Promise((resolve, reject) => {
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = () => {
      const db = req.result;
      if (!db.objectStoreNames.contains(STORE_ICONS)) {
        db.createObjectStore(STORE_ICONS, { keyPath: 'id' });
      }
    };
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
  return dbPromise;
}

// 保存 Blob
export async function saveIconBlob(id: string, blob: Blob): Promise<void> {
  const db = await openDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_ICONS, 'readwrite');
    const store = tx.objectStore(STORE_ICONS);
    const req = store.put({ id, blob });
    req.onsuccess = () => resolve();
    req.onerror = () => reject(req.error);
    tx.onabort = () => reject(tx.error);
  });
}
```

### 技术选型原因

- **IndexedDB vs localStorage**：localStorage 容量通常仅 5-10MB 且只能存字符串；IndexedDB 容量可达数百 MB 甚至数 GB，原生支持 Blob 二进制存储，适合存放图片文件。
- **`keyPath: 'id'` 主键策略**：以图标 ID 作为主键，支持按 ID 快速 O(1) 查找和覆盖更新（`put` 而非 `add`）。
- **单例 Promise 缓存**：`dbPromise` 缓存数据库连接，避免重复 `indexedDB.open()` 开销。
- **事务监听**：同时监听 `req.onerror` 和 `tx.onabort`，确保事务级错误也能被捕获。
- **Promise 封装**：将 IndexedDB 的事件回调 API 转为现代 Promise。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| 用户禁用 IndexedDB / 隐私模式 | 浏览器无痕模式下 IndexedDB 可能不可用 | `openDB` reject，Store 层捕获并提示 Toast |
| 磁盘配额不足 | 本地存储空间已满 | `QuotaExceededError` 错误，提示用户清理空间 |
| 数据库版本冲突 | 未来升级 DB_VERSION 时迁移逻辑缺失 | 当前 `onupgradeneeded` 仅创建 store，无迁移 |
| 事务被中止 | 存储过程中页面关闭或崩溃 | `tx.onabort` 触发 reject |
| Blob 在 IndexedDB 中序列化失败 | 极少数浏览器对 Blob 存储有 Bug | 降级方案：存储 ArrayBuffer + type 字段 |
| 数据被用户手动清除 | 开发者工具或浏览器设置清除站点数据 | 读取时返回 null，UI 层显示加载失败提示 |

---

## 环节八：提取元数据保存到 localStorage

### 代码位置

- 元数据转换：[utils/index.ts#L84-L87](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/index.ts#L84-L87)
- Store 层保存逻辑：[useAppStore.ts#L114-L133](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/store/useAppStore.ts#L114-L133)
- localStorage 写入：[useAppStore.ts#L84-L93](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/store/useAppStore.ts#L84-L93)
- 类型定义：[types/index.ts#L1-L12](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/types/index.ts#L1-L12)

### 技术实现

```typescript
// types/index.ts
export interface IconMeta {
  id: string;
  name: string;
  originalName: string;
  width: number;
  height: number;
  addedAt: number;
}

export interface IconItem extends IconMeta {
  dataUrl: string;  // 仅内存中存在，不落盘
}

// utils/index.ts —— 剥离 dataUrl，仅保留元数据
export function iconItemToMeta(item: IconItem): IconMeta {
  const { dataUrl: _, ...meta } = item;
  return meta;
}

// useAppStore.ts
const STORAGE_KEY = 'css-sprite-tool-data';

function saveToStorage(projects: Project[], icons: IconMeta[]): boolean {
  try {
    const payload = JSON.stringify({ projects, icons });
    localStorage.setItem(STORAGE_KEY, payload);
    return true;
  } catch (e) {
    toastHandlers.showError('本地存储失败，浏览器存储空间可能已满');
    return false;
  }
}

// addIcons 流程
addIcons: async (items) => {
  if (items.length === 0) return;
  const metas: IconMeta[] = items.map(iconItemToMeta);

  try {
    for (const item of items) {
      await saveIconDataUrl(item.id, item.dataUrl);  // 二进制存 IndexedDB
    }
  } catch (e) {
    toastHandlers.showError('保存图片到本地数据库失败');
    throw e;
  }

  set((state) => {
    const newIcons = [...state.icons, ...metas];
    saveToStorage(state.projects, newIcons);  // 元数据存 localStorage
    return { icons: newIcons };
  });
}
```

### 存储分层设计

```
┌─────────────────────────────────────────────┐
│           localStorage (5-10MB)             │
│  ┌───────────────────────────────────────┐  │
│  │ Key: css-sprite-tool-data             │  │
│  │ Value: JSON {                         │  │
│  │   projects: [ {id, name, iconIds} ],  │  │
│  │   icons: [ {id, name, width, ...} ]   │  │  ← 元数据（小体积）
│  │ }                                     │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│           IndexedDB (数百 MB ~ GB 级)       │
│  ┌───────────────────────────────────────┐  │
│  │ Store: icons (keyPath: id)            │  │
│  │ Records:                              │  │
│  │   { id: "lq8x2k0a", blob: Blob }      │  │  ← 二进制图片数据（大体积）
│  │   { id: "a3b7c1d2", blob: Blob }      │  │
│  │   ...                                 │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 技术选型原因

- **冷热数据分离**：元数据（图标名、尺寸、ID 等）体积小、访问频繁，适合 localStorage；图片二进制体积大、访问相对不频繁，适合 IndexedDB。
- **localStorage 同步读取**：应用启动时 `loadFromStorage()` 同步读取元数据，无需异步等待 IndexedDB，首屏渲染更快。
- **JSON 序列化**：结构清晰，便于调试和手动恢复，元数据量小（单个图标约 100 字节），序列化开销可忽略。
- **IconMeta vs IconItem 类型分层**：TypeScript 类型系统确保 dataUrl 不会意外被序列化到 localStorage，避免空间浪费。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| localStorage 配额超限 | 图标项目极多，JSON 超过 5-10MB | `setItem` 抛出异常，Toast 提示用户清理 |
| JSON 序列化失败 | 元数据中包含循环引用或特殊字符 | 当前结构扁平（无嵌套对象），不会发生 |
| 用户手动篡改 localStorage | 删除或修改 JSON 内容 | `loadFromStorage` try/catch 包裹，损坏则回退空数据 |
| 元数据与 IndexedDB 数据不一致 | IndexedDB 数据被清除但元数据仍在 | 读取时 `getIconDataUrl` 返回 null，UI 显示加载失败 |
| 隐私模式下 localStorage 不可用 | 某些浏览器无痕模式限制 | `loadFromStorage` 返回空数组，功能降级为纯内存 |
| 数据版本不兼容 | 未来升级数据结构但旧数据未迁移 | 建议添加 `version` 字段，当前暂无 |

---

## 环节九：图标库列表渲染展示

### 代码位置

- 从 IndexedDB 加载图标数据：[useAppStore.ts#L257-L291](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/store/useAppStore.ts#L257-L291)
- Blob 转回 DataURL：[utils/db.ts#L67-L71](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/db.ts#L67-L71) 和 [utils/db.ts#L117-L124](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/utils/db.ts#L117-L124)
- Library 页面渲染：[Library.tsx#L227-L346](file:///d:/code/gy/45/45-未归类-21/45-未归类-21/src/pages/Library.tsx#L227-L346)

### 技术实现

```typescript
// useAppStore.ts —— 单个图标加载
getIconItem: async (meta) => {
  try {
    const dataUrl = await getIconDataUrl(meta.id);
    if (!dataUrl) {
      toastHandlers.showWarning(`图标 "${meta.name}" 数据缺失`);
      return null;
    }
    return { ...meta, dataUrl };
  } catch (e) {
    toastHandlers.showError(`加载图标 "${meta.name}" 失败`);
    return null;
  }
}

// useAppStore.ts —— 批量加载项目图标
getIconsInProject: async (projectId) => {
  const state = get();
  const project = state.projects.find((p) => p.id === projectId);
  if (!project) return { items: [], total: 0, loaded: 0, failed: 0 };
  const metaMap = new Map(state.icons.map((i) => [i.id, i]));
  const metas = project.iconIds
    .map((id) => metaMap.get(id))
    .filter((m): m is IconMeta => !!m);

  const items: IconItem[] = [];
  let failed = 0;
  for (const meta of metas) {
    const item = await get().getIconItem(meta);
    if (item) items.push(item);
    else failed++;
  }
  return { items, total: metas.length, loaded: items.length, failed };
}

// utils/db.ts —— Blob → DataURL
export function blobToDataUrl(blob: Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = reject;
    reader.readAsDataURL(blob);
  });
}
```

### 渲染流程

```
用户选择项目
       ↓
useEffect 触发 getIconsInProject(projectId)
       ↓
从 localStorage(内存状态) 读取 IconMeta[] 列表
       ↓
串行遍历每个 Meta:
  ├─ getIconBlob(id) → 从 IndexedDB 取 Blob
  ├─ blobToDataUrl(blob) → FileReader 转回 DataURL
  └─ 组装 IconItem = Meta + DataURL
       ↓
统计 loaded / failed 数量
       ↓
setProjectIcons(items) → 更新 React state
       ↓
renderContentArea() → 网格布局渲染 <img src={icon.dataUrl}>
```

### 技术选型原因

- **按需读取 Blob**：不一次性将所有图片加载到内存，仅在用户切换到对应项目时才读取，节省内存。
- **`FileReader.readAsDataURL`**：与上传时的读取方式对称，确保 DataURL 格式一致，可直接用于 `<img>`。
- **串行加载而非 `Promise.all` 并行**：避免同时触发大量 IndexedDB 读取和 FileReader 实例，减少主线程阻塞和内存峰值（特别是图标数量多时）。
- **失败容错统计**：`loaded/failed/total` 三个计数器，UI 层可精确展示加载状态，部分失败不影响整体渲染。
- **渐进式加载体验**：`isLoading` 状态显示骨架屏，加载完成后展示内容；全部失败时提供"重试加载"和"重新上传"按钮。

### 可能的失败点

| 失败场景 | 原因 | 处理方式 |
|---------|------|---------|
| IndexedDB 记录不存在 | 数据被清除或 ID 错误 | `getIconBlob` 返回 null，标记为 failed |
| Blob 读取失败 | Blob 数据损坏或浏览器异常 | `FileReader.onerror` 触发 reject，标记 failed |
| 加载过程中切换项目 | 前一次加载的结果覆盖新项目 | `useEffect` cleanup 中设置 `cancelled = true` 丢弃旧结果 |
| 大量图标加载慢 | 数百个图标串行读取 | 显示加载动画和进度提示（`loaded/total`） |
| 内存占用过高 | 大量 DataURL 长字符串驻留内存 | 切换项目后旧 state 被 GC 回收；长期运行建议虚拟滚动 |
| 图片损坏无法解码 | Blob 内容不完整 | `<img>` 标签会显示为破损图标，浏览器默认占位 |

---

## 总结：数据流转架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户交互层 (UI)                              │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────────┐   │
│  │ 点击上传按钮  │    │ 拖拽文件区域  │    │ 图标库网格渲染 <img> │   │
│  └──────┬───────┘    └──────┬───────┘    └──────────▲──────────┘   │
└─────────┼───────────────────┼───────────────────────┼──────────────┘
          │                   │                       │
          ▼                   ▼                       │
┌─────────────────────────────────────────────────────────────────────┐
│                      工具函数层 (utils)                              │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────────┐   │
│  │ isImageFile  │───▶│ fileToDataUrl│───▶│    getImageSize     │   │
│  │ (MIME校验)   │    │ (FileReader) │    │  (HTMLImageElement) │   │
│  └──────────────┘    └──────────────┘    └──────────┬──────────┘   │
│                                                     │              │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────────┐   │
│  │ dataUrlToBlob│◀───│  generateId  │◀───│ createIconItemFrom..│   │
│  │ (atob解码)   │    │ (时间戳+随机) │    │  (组装 IconItem)    │   │
│  └──────┬───────┘    └──────────────┘    └─────────────────────┘   │
└─────────┼───────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       状态管理层 (Store)                             │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ addIcons(items):                                              │  │
│  │   1. items.map(iconItemToMeta) → 剥离 dataUrl                │  │
│  │   2. for each: saveIconDataUrl(id, dataUrl) → IndexedDB      │  │
│  │   3. set({ icons: [...state.icons, ...metas] })              │  │
│  │   4. saveToStorage(projects, icons) → localStorage           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ getIconsInProject(projectId):                                 │  │
│  │   1. 从 state 读取 IconMeta[] 列表                            │  │
│  │   2. for each meta: getIconBlob → blobToDataUrl → IconItem   │  │
│  │   3. 返回 { items, total, loaded, failed }                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────┬───────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        持久化存储层 (Storage)                        │
│  ┌─────────────────────────┐    ┌────────────────────────────────┐ │
│  │      localStorage       │    │          IndexedDB              │ │
│  │  Key: css-sprite-tool.. │    │  DB: sprite-lab-db v1          │ │
│  │  {                      │    │  Store: icons (keyPath: id)    │ │
│  │    projects: [...],     │    │  Records:                      │ │
│  │    icons: [Meta, Meta..]│    │    { id, blob: Blob }          │ │
│  │  }                      │    │    { id, blob: Blob }          │ │
│  └─────────────────────────┘    └────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键技术决策对照表

| 决策点 | 选择方案 | 备选方案 | 选择理由 |
|-------|---------|---------|---------|
| ID 生成 | 时间戳+随机数 Base36 | UUID / nanoid | 无依赖、时间有序、足够唯一 |
| 图片上传 | HTML5 File API | Dropzone / uppy | 轻量、无依赖、满足需求 |
| 格式校验 | MIME 白名单正则 | 文件头魔数检测 | 实现简单、99% 场景够用 |
| 中间格式 | DataURL (Base64) | ObjectURL | 可存入 localStorage、可序列化传递 |
| 大体积存储 | IndexedDB (Blob) | localStorage Base64 | 容量大 100x、存储效率高 33% |
| 元数据存储 | localStorage (JSON) | IndexedDB 单独 Store | 同步读取、首屏快、结构简单 |
| 状态管理 | Zustand | Redux / MobX | 极简 API、TS 友好、无样板代码 |
| 加载策略 | 串行按需加载 | 并行 Promise.all | 避免阻塞、内存可控 |

---

## 开发说明

### 安装依赖

```bash
npm install
```

### 启动开发服务器

```bash
npm run dev
```

### 构建生产版本

```bash
npm run build
```

### 运行测试

```bash
npm run test
```

### 代码检查

```bash
npm run lint
```
