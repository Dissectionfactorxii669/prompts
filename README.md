# Prompts & Skills

A collection of prompts, skills, and agent rules for AI-powered development workflows.

## Structure

```
claude-code/
  skills/         # Claude Code custom skills
  rules/          # Global / project-level agent rules
```

## Skills

### coding-agents

Claude Code 作为调度者，协调 Codex CLI（后端）和 Gemini CLI（前端）完成开发任务的协作规范。

- 角色分工与任务分配决策树
- 各 CLI 的常用命令速查
- Session continue vs 新 task 的判断标准
- Prompt 编写原则与模板

## License

MIT
