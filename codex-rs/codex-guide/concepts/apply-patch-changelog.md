# Apply-Patch 更新日志与变更摘要

## 最近更新 (2025-08-23)

### 1. Lenient 解析模式 (GPT-4.1 兼容性)

**新增功能**：
- 自动识别并处理 heredoc 包装 (`<<EOF`, `<<'EOF'`, `<<"EOF"`)
- 兼容 GPT-4.1 将 heredoc 作为参数传递的行为
- 保持补丁体语法严格校验的同时，放宽外层传参方式

**实现位置**：
- [`apply-patch/src/parser.rs:115-152`](apply-patch/src/parser.rs:115) - ParseMode 枚举定义
- [`apply-patch/src/parser.rs:199-220`](apply-patch/src/parser.rs:199) - check_patch_boundaries_lenient 函数

**使用示例**：
```bash
# GPT-4.1 常见方式（自动兼容）
["apply_patch", "<<'EOF'\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]

# 传统方式（无需特殊处理）
["bash", "-lc", "apply_patch <<'EOF'\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]
```

### 2. 增强的错误处理

**改进内容**：
- 详细的错误信息包含具体行号
- 分类错误类型：ParseError, ComputeReplacements, IoError
- 清晰的上下文匹配失败提示

**错误类型定义**：
- [`apply-patch/src/parser.rs:49-55`](apply-patch/src/parser.rs:49) - ParseError 枚举
- [`apply-patch/src/lib.rs:27-36`](apply-patch/src/lib.rs:27) - ApplyPatchError 枚举

### 3. 模型特定行为文档化

**GPT-4.1 特殊处理**：
- 自动注入工具使用说明文档
- 基于模型家族的动态配置
- 条件性工具指令拼接

**配置位置**：
- [`core/src/client_common.rs:44`](core/src/client_common.rs:44) - Prompt::get_full_instructions()
- [`core/src/model_family.rs:68`](core/src/model_family.rs:68) - find_family_for_model()

### 4. 算法优化

**seek_sequence 算法特性**：
- Unicode 字符标准化处理
- 模糊匹配常见标点符号变体
- 高效的模式搜索性能
- 支持部分匹配和重试机制

**实现位置**：
- [`apply-patch/src/seek_sequence.rs`](apply-patch/src/seek_sequence.rs) - 核心算法实现

### 5. 文件操作原子性

**操作保证**：
- 添加文件：目录创建→文件写入
- 更新文件：读取→修改→写入
- 移动文件：复制→验证→删除源
- 删除文件：直接删除（不可逆）

## 向后兼容性

### 完全兼容
- 所有现有补丁格式继续支持
- 传统 heredoc 使用方式无需修改
- 错误处理接口保持向后兼容

### 新增功能
- 自动 heredoc 检测（无需用户干预）
- 更详细的错误诊断信息
- 模型特定的优化处理

## 使用建议

### 对于 GPT-4.1 用户
- 可以直接使用 heredoc 语法，系统会自动处理
- 推荐使用 shell 语义以获得最佳兼容性
- 遵循注入的工具使用说明

### 对于其他模型
- 继续使用标准补丁格式
- 可选择 Strict 或 Lenient 模式
- 保持现有使用习惯

## 测试覆盖

### 新增测试用例
- heredoc 包装的各种变体测试
- GPT-4.1 兼容性测试
- 错误处理边界情况
- Unicode 字符处理测试

### 测试位置
- [`apply-patch/src/parser.rs:560-641`](apply-patch/src/parser.rs:560) - Lenient 模式测试
- [`apply-patch/src/lib.rs:690-1335`](apply-patch/src/lib.rs:690) - 集成测试

## 文档更新

### 主要更新内容
- 新增模型特定行为章节
- 详细说明 Lenient 解析模式
- 提供 heredoc 使用最佳实践
- 补充错误处理指南

### 文档位置
- [`codex-guide/concepts/apply-patch.md`](codex-guide/concepts/apply-patch.md) - 主要文档
- [`apply-patch/apply_patch_tool_instructions.md`](apply-patch/apply_patch_tool_instructions.md) - 工具指令

---

*更新日期：2025-08-23*
