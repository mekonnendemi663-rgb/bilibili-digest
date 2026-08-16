# Digest for Bilibili — Agent 指南

B 站视频学习侧边栏扩展（Chrome MV3）。架构源自 [youtube-digest](https://github.com/zarazhangrui/youtube-digest)，字幕与 AI 能力针对 B 站接口做了适配。

## 常用命令

```bash
npm test              # 运行 tests/*.test.js（Node 内置 test runner）
npm run package       # 打包 dist/bilibili-digest.zip
bash scripts/normalize-eol.sh   # 统一换行符（发布前可选）
```

修改代码后务必跑 `npm test`。本项目没有 linter 脚本，测试即验收标准。

## 架构概览

```
manifest.json
├── background.js          # Service worker：消息路由、WBI 签名、B 站 API、LLM 调用、缓存
├── content.js             # 注入 B 站播放页：侧边栏入口、seek、笔记快捷键
├── sidepanel.js/html/css  # 侧边栏 UI：字幕 / 概览 / 笔记
├── options.js/html/css    # 设置页：AI 密钥、模型参数
├── settings.js            # 设置读写与校验
└── lib/
    ├── bili-api.js        # B 站字幕与视频信息
    ├── wbi.js             # WBI 签名
    ├── transcript.js      # 字幕分段与语义段落重组
    ├── cache.js           # chrome.storage.local 缓存
    ├── ai.js              # 提示词解析、JSON 提取、分块合并
    ├── ai-provider.js     # OpenAI 兼容 API 调用
    └── concurrency.js     # 并发批次控制
prompts/                   # AI 提示词 markdown（background 按节读取）
tests/                     # 单元测试 + sidepanel 集成测试（vm 桩 DOM/chrome）
```

### 消息流

侧边栏 / 内容脚本通过 `chrome.runtime.sendMessage` 与 `background.js` 通信。常见 action：

| action | 用途 |
|--------|------|
| `fetchTranscript` | 拉取字幕、视频信息、缓存 |
| `analyzeTranscript` | AI 章节概览 |
| `translateSegments` | 分段翻译 |
| `polishSegments` | 顺句（中文标点修复） |
| `explainSelection` | 划词解释 |
| `getNotes` / `saveNote` / `deleteNote` | 笔记 CRUD |

密钥与缓存仅存在于扩展可信上下文（`chrome.storage.local` + `TRUSTED_CONTEXTS`）。

## 编码约定

- **语言**：用户可见文案用中文；代码注释与 commit 风格与现有文件保持一致。
- **模块**：`lib/*.js` 用 IIFE 挂到 `BILI_*` 全局，并在末尾 `module.exports` 供测试 require；浏览器端通过 `<script>` 或 `importScripts` 加载。
- **DOM 安全**：字幕、笔记等外部内容一律 `textContent` 写入，禁止拼 HTML 字符串。
- **状态**：`sidepanel.js` 的 `state` 对象集中管理；换视频时用 `bvid` 校验异步结果，避免旧请求覆盖新界面。
- **提示词**：放在 `prompts/*.md`，用 `## 小节名` + 围栏代码块；变量用 `{name}` 占位，由 `BILI_AI.extractPromptSection` 替换。
- **测试**：新增 lib 逻辑写独立单元测试；UI 行为优先在 `tests/sidepanel.test.js` 用 vm 加载真实 `sidepanel.js` + DOM/chrome 桩，不要复制业务逻辑到测试里。

## 字幕相关要点

- **三视图**：`original` / `translated` / `bilingual`，中文字幕与外文字幕都支持；翻译方向由 `background` 根据字幕语种决定。
- **顺句**：仅中文字幕（`state.isChinese`）；开启后「原文」指顺句稿（`state.polished`）。
- **复制 / 导出**：跟随当前视图——看到什么导出什么。复制为纯文本（`transcriptAsText`），导出为 Markdown（`transcriptAsMarkdown`）。
- **分段**：`lib/transcript.js` 的 `groupTranscriptEntries` 把 ASR 条目重组为可读段落；CJK 与拉丁文使用不同阈值。

## 修改时的注意点

- 不要安装 Grok CLI 或引入与本项目无关的重型依赖；扩展保持零构建、纯 JS。
- `manifest.json` 变更后更新 `tests/manifest.test.js`（若有字段断言）。
- 商店文案见 `STORE-LISTING.md`、`README.md`；功能描述与实际行为保持一致。
- Chrome 116+ / Edge 116+（侧边栏 API）；不要用仅 Chrome 独有的 per-tab `sidePanel.setOptions` 逻辑。
- AI 调用有超时与重试；部分批次失败应降级展示已完成部分，而非整页报错。

## 验证清单

- [ ] `npm test` 全部通过
- [ ] 若改 UI 文案，同步 README / STORE-LISTING
- [ ] 若改导出格式，确认 `.md` 在 Obsidian / 记事本中可读，双语模式译文用引用块
