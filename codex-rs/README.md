# Codex

对 [openai codex](https://github.com/openai/codex) 的一个fork版本， codex是一个类似claude code的本地ai code agent。区别是，claude code是闭源，而codex开源。更大的区别是，claude code非常好用，非常智能————即使搭配上非claude模型（如：kimi-k2）。而codex即使用自家的gpt5，也显得呆呆的

## Purpose

尽管如此，但codex整体的设计比较简洁、架构可扩展性强，非常方便我们进行定制和扩展。并且，可以作为个人助手工具持续迭代，比如：

* 对接公司内的mcp服务
* 针对内部LLM优化内置prompt
* 内部更多流程工具
* 支持memory能力
* ……
