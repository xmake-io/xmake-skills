<div align="center">
  <a href="https://xmake.io">
    <img width="160" height="160" src="https://xmake.io/assets/img/logo.png">
  </a>

  <h1>xmake-skills</h1>

  <div>
    <a href="https://github.com/xmake-io/xmake-skills/blob/main/LICENSE.md">
      <img src="https://img.shields.io/github/license/xmake-io/xmake-skills.svg?colorB=f48041&style=flat-square" alt="license" />
    </a>
    <a href="https://discord.gg/xmake">
      <img src="https://img.shields.io/badge/chat-on%20discord-7289da.svg?style=flat-square" alt="Discord" />
    </a>
  </div>

  <b>面向 Xmake 的 Agent Skills</b><br/>
  <i>一套用于让 AI 助手正确使用 Xmake 的 Claude Agent Skills 集合。</i><br/>
</div>

## 简介 ([English](/README.md))

**xmake-skills** 是一套为 [Xmake](https://xmake.io) 设计的 [Agent Skills](https://www.anthropic.com/news/agent-skills)，Xmake 是一个基于 Lua 的跨平台构建工具。这些 Skills 为 AI 编程助手（例如 Claude Code）提供必要的知识，使其能够正确地创建、配置、构建、调试和打包 Xmake 工程。

每个 Skill 封装了一块聚焦的 Xmake 知识 —— 从编写 `xmake.lua` 到集成 C/C++ 依赖包 —— 这样 AI Agent 就能生成符合最佳实践、可直接运行的 Xmake 配置，而不是靠猜测。

## 背景

大语言模型通常"知道" Xmake，但经常在细节上出错：API 过时、选项名称错误、包语法不正确。Skills 通过在 Agent 真正处理 Xmake 相关任务时，按需加载合适的文档和示例进入上下文，来解决这一问题。

## 使用

将本仓库克隆到你的 Agent 的 Skills 目录中，或者直接在 Agent 中指向本仓库。具体安装和启用 Skills 的方法，请参考对应 Agent 平台的文档。

```bash
git clone https://github.com/xmake-io/xmake-skills.git
```

## 贡献

非常欢迎贡献。如果你想新增一个 Skill 或改进已有 Skill，请提交 Issue 或 Pull Request。

## 相关资源

- Xmake: https://github.com/xmake-io/xmake
- 文档: https://xmake.io
- 社区: https://discord.gg/xmake

## 许可证

采用 Apache License 2.0 许可证。详情请见 [LICENSE.md](/LICENSE.md)。
