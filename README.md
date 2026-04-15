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

  <b>Agent Skills for Xmake</b><br/>
  <i>A collection of Claude Agent Skills that teach AI assistants how to use Xmake effectively.</i><br/>
</div>

## Introduction ([中文](/README_zh.md))

**xmake-skills** is a collection of [Agent Skills](https://www.anthropic.com/news/agent-skills) for [Xmake](https://xmake.io), the cross-platform build utility based on Lua. These skills give AI coding assistants (such as Claude Code) the knowledge they need to create, configure, build, debug, and package Xmake projects correctly.

Each skill packages up focused Xmake know-how — from writing `xmake.lua` to integrating C/C++ packages — so AI agents can produce idiomatic, working Xmake configurations instead of guessing.

## Why

Large language models often know *about* Xmake but get the details wrong: outdated APIs, wrong option names, broken package syntax. Skills fix this by loading just the right documentation and examples into the agent's context when it is actually working on Xmake.

## Usage

Clone this repository into your agent's skills directory, or point your agent at it directly. Refer to your agent platform's documentation for how to install and enable skills.

```bash
git clone https://github.com/xmake-io/xmake-skills.git
```

## Contributing

Contributions are very welcome. If you want to add a new skill or improve an existing one, please open an issue or pull request.

## Resources

- Xmake: https://github.com/xmake-io/xmake
- Documentation: https://xmake.io
- Community: https://discord.gg/xmake

## License

Licensed under the Apache License, Version 2.0. See [LICENSE.md](/LICENSE.md) for details.
