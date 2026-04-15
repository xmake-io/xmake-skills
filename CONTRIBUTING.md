# Contributing

If you discover issues, have ideas for improvements, or want to contribute a new skill, please report them to the [issue tracker][1] of the repository or submit a pull request. Please try to follow these guidelines when you do so.

## Issue reporting

* Check that the issue has not already been reported.
* Check that the issue has not already been fixed in the latest code (a.k.a. `main`).
* Be clear, concise, and precise in your description of the problem.
* Open an issue with a descriptive title and a summary in grammatically correct, complete sentences.
* Include any relevant code, `xmake.lua` snippets, or agent transcripts in the issue.

## Pull requests

* Please update your local branch to the latest before submitting a pull request to ensure no merge conflicts.
* Use a topic branch to easily amend a pull request later, if necessary.
* Write good commit messages. Please use English for commit messages to standardize the log format.
* Use the same conventions as the rest of the project.
* When adding or modifying a skill, make sure the skill's `SKILL.md` description stays focused and accurate — it is what the agent reads to decide whether to load the skill.
* Keep skill content grounded in real, verified Xmake behavior. Do not include invented APIs or options.

## Adding a new skill

Each skill lives in its own directory and follows the [Agent Skills](https://www.anthropic.com/news/agent-skills) format:

```
skills/<skill-name>/
  SKILL.md           # required: frontmatter + instructions
  references/        # optional: supporting docs the skill can load
  scripts/           # optional: helper scripts
```

When proposing a new skill, please include:

* A short rationale in the PR describing what Xmake task the skill helps with and why an agent needs it.
* At least one concrete example of a prompt the skill should handle correctly.

## Financial contributions

If you want to support Xmake but are unable to contribute code, you can also support the community through [financial contributions](https://xmake.io/about/sponsor).

---

# 贡献代码

如果你发现问题、有改进想法，或者想要贡献一个新的 Skill，可以在 [issues][1] 上反馈，或者发起一个 Pull Request。请在贡献时遵循以下指南。

## 问题反馈

* 确认这个问题没有被反馈过。
* 确认这个问题最近还没有被修复，请先检查 `main` 的最新提交。
* 请清晰详细地描述你的问题。
* 请使用语法正确、完整的句子，开启一个带有描述性标题和摘要的 issue。
* 如果涉及具体代码、`xmake.lua` 片段或 Agent 对话记录，请一并附上。

## 提交代码

* 请先更新你的本地分支到最新，再提交 Pull Request，确保没有合并冲突。
* 如果需要，使用 topic 分支以便稍后轻松修改 Pull Request。
* 编写友好可读的提交信息。为了规范化提交日志，commit 信息请使用英文描述。
* 请使用与工程相同的代码规范。
* 新增或修改 Skill 时，请确保 `SKILL.md` 的描述准确且聚焦 —— Agent 就是根据它来决定是否加载该 Skill 的。
* Skill 内容必须基于真实、可验证的 Xmake 行为，不要编造 API 或选项。

## 新增 Skill

每个 Skill 独立一个目录，遵循 [Agent Skills](https://www.anthropic.com/news/agent-skills) 格式：

```
skills/<skill-name>/
  SKILL.md           # 必需：frontmatter + 指令
  references/        # 可选：可按需加载的补充文档
  scripts/           # 可选：辅助脚本
```

在提交新 Skill 时，请在 PR 中包含：

* 简短的理由，说明这个 Skill 解决什么 Xmake 场景，以及 Agent 为什么需要它。
* 至少一个该 Skill 应该能正确处理的具体提示词示例。

## 资金赞助

如果你想支持 Xmake 但无法贡献代码，也可以通过[资金赞助](https://xmake.io/about/sponsor)来支持社区的进一步发展。

[1]: https://github.com/xmake-io/xmake-skills/issues
