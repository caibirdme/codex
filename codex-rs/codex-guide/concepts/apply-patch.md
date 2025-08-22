# Apply-Patch 机制详解

## 概述

Apply-patch 是 Codex CLI 的核心文件编辑机制，它提供了一种基于上下文的补丁应用方式，而不是传统的行号-based diff。这种设计使得代码修改更加稳定和可靠，特别是在自动化环境中。

## 语法规范

### 基本结构

每个补丁必须包含开始和结束标记：

```bash
*** Begin Patch
[一个或多个文件操作]
*** End Patch
```

### 文件操作类型

#### 1. 添加文件 (Add File)
```bash
*** Add File: path/to/file.py
+第一行内容
+第二行内容
```

#### 2. 删除文件 (Delete File)
```bash
*** Delete File: path/to/obsolete.py
```

#### 3. 更新文件 (Update File)
```bash
*** Update File: path/to/file.py
[可选的移动操作]
[一个或多个修改块]
```

#### 4. 移动文件 (Move File)
```bash
*** Update File: path/to/source.py
*** Move to: path/to/destination.py
[修改块]
```

### 修改块语法

#### 基本修改块
```bash
@@ [上下文信息]
 上下文行（保持不变）
-要删除的行
+要添加的行
 更多上下文行
```

#### 嵌套上下文
```bash
@@ class MyClass
@@     def method():
-        old_implementation
+        new_implementation
```

#### 文件末尾添加
```bash
@@
+在文件末尾添加的行
*** End of File
```

## 上下文匹配机制

### 工作原理

Apply-patch 使用上下文匹配而不是行号来定位修改位置：

1. **解析上下文**：提取 `@@` 后面的上下文信息
2. **模式搜索**：使用 `seek_sequence` 算法在文件中查找匹配的代码模式
3. **精确匹配**：找到匹配位置后应用修改
4. **容错处理**：支持模糊匹配和 Unicode 字符标准化

### 上下文类型

#### 1. 空上下文 (`@@`)
使用默认的3行上下文进行匹配

#### 2. 显式上下文 (`@@ context`)
```bash
@@ def my_function():
```

#### 3. 嵌套上下文
```bash
@@ class MyClass
@@     def specific_method():
```

## 实际示例

### 简单修改
```bash
*** Begin Patch
*** Update File: example.py
@@
-print("Hello")
+print("Hello, World!")
*** End Patch
```

### 多个修改块
```bash
*** Begin Patch
*** Update File: utils.py
@@ def calculate():
-    return 0
+    return 1
@@ def validate():
-    return False  
+    return True
*** End Patch
```

### 带上下文的复杂修改
```bash
*** Begin Patch
*** Update File: app/models.py
@@ class User:
@@     def get_name():
-        return self.name
+        return self.name.title()
@@ class Product:
@@     def get_price():
-        return self.price
+        return round(self.price, 2)
*** End Patch
```

## 最佳实践

### 1. 上下文选择
- 使用具体的类名、函数名作为上下文
- 避免使用过于通用的上下文（如 `@@ def`）
- 嵌套上下文提供更精确的定位

### 2. 错误处理
- 确保上下文在文件中唯一存在
- 测试补丁在应用前是否能够正确定位
- 使用 `*** End of File` 明确标识文件末尾操作

### 3. 性能考虑
- 单个补丁中包含多个相关修改
- 避免不必要的文件操作
- 使用相对路径而非绝对路径

## 与传统 diff 的区别

| 特性 | 传统 Diff | Apply-Patch |
|------|-----------|-------------|
| 定位方式 | 行号 | 上下文匹配 |
| 稳定性 | 行号易变 | 上下文稳定 |
| 可读性 | 需要行号上下文 | 自描述性 |
| 容错性 | 低（行号变化即失败） | 高（模式匹配） |

## 常见问题解答

### Q: 为什么不需要行号？
A: 行号在代码修改过程中会发生变化，基于上下文的匹配更加稳定可靠。

### Q: @@ 符号可以嵌套吗？
A: 是的，支持多层嵌套上下文提供更精确的定位。

### Q: 如果上下文不唯一怎么办？
A: 系统会尝试找到第一个匹配项，建议使用更具体的上下文。

### Q: 支持正则表达式吗？
A: 不支持正则表达式，使用精确的字符串匹配。

## 实现细节

### 核心算法
Apply-patch 使用 `seek_sequence` 算法进行模式匹配，该算法能够：

1. 在文件中查找连续的代码序列
2. 处理 Unicode 字符的标准化
3. 支持模糊匹配常见标点符号变体
4. 提供高效的搜索性能

### 错误处理
- 上下文未找到时提供清晰的错误信息
- 支持部分匹配和重试机制
- 详细的日志输出帮助调试

## 参考资源

- [`apply-patch/src/lib.rs`](../apply-patch/src/lib.rs) - 核心实现
- [`apply-patch/src/parser.rs`](../apply-patch/src/parser.rs) - 语法解析器
- [`apply-patch/apply_patch_tool_instructions.md`](../apply-patch/apply_patch_tool_instructions.md) - 工具指令

---

*本文档最后更新于: 2025-08-22*

## 模型特定行为与兼容性（GPT‑4.1、Lenient 解析与工具说明）

本节补充解释为什么 apply_patch 的「额外使用说明」只对特定模型注入（目前是 GPT‑4.1），以及解析器为何引入 Lenient 模式来兼容该模型在工具调用上的行为差异。

- 说明注入位置与条件
  - 模型请求负载中的 base instructions 会根据模型家族特征选择性地拼接 apply_patch 使用说明文档。逻辑位于 [Prompt::get_full_instructions()](core/src/client_common.rs:44)。仅当模型家族标记 needs_special_apply_patch_instructions=true 时，才会将 [`APPLY_PATCH_TOOL_INSTRUCTIONS`](apply-patch/src/lib.rs:23) 文本附加到指令末尾。
  - 哪些模型需要该说明由模型家族解析决定，参见 [find_family_for_model()](core/src/model_family.rs:68)。目前针对以 "gpt‑4.1" 开头的模型，会设置 needs_special_apply_patch_instructions=true（见同文件 92–96 行分支）。
  - 测试覆盖了这一行为，见 [tests::get_full_instructions_no_user_content()](core/src/client_common.rs:144)。

- 为什么是 GPT‑4.1 需要特殊说明
  - GPT‑4.1 在调用本地命令（local_shell）时，常构造如下数组参数：
    - ["apply_patch", "&lt;&lt;'EOF' ... EOF\n"]（将 heredoc 作为「参数」传递给可执行程序）
  - 然而 local_shell 的实现并不会通过交互式 shell（如 bash）来解释 heredoc，而是直接 execvpe（或等效）运行命令数组。把 "<<'EOF' ... EOF" 当作 argv 传入可执行文件并不具备 shell 语义，按常规会失败。
  - 为了尽可能兼容 GPT‑4.1 的这种「将 heredoc 当作参数传递」的用法，解析器引入了 Lenient 模式，自动识别并剥离 heredoc 标记，将中间正文作为真实 patch 文本继续解析。详见 [ParseMode](apply-patch/src/parser.rs:115) 及其注释（119–152 行对 heredoc 兼容的动机与示例有详细阐述）。

- 解析器的 Strict vs Lenient 策略
  - 全局开关 [PARSE_IN_STRICT_MODE](apply-patch/src/parser.rs:47) 当前为 false，因此入口 [parse_patch()](apply-patch/src/parser.rs:106) 默认走 Lenient。
  - 核心兼容逻辑在 [check_patch_boundaries_lenient()](apply-patch/src/parser.rs:199)：当首行是 "<<EOF" 或带引号的 "<<'EOF'" / "<<"EOF"" 且末行以 "EOF" 结束时，去掉外层 heredoc，再以严格规范校验内部是否以 "*** Begin Patch" 开始、以 "*** End Patch" 结束，并继续进行完整的语法解析。
  - 重要的是：Lenient 只放宽「外层传参方式」（heredoc 包裹方式），对补丁体本身的结构、hunk 语法、上下文标记仍保持完整的结构化校验。语法错误会返回具体错误（例如开头/结尾标记缺失、空 Update hunk、非 ' + -  ' 前缀的行等），参考多处断言用例于 [parse_patch_text()](apply-patch/src/parser.rs:154) 周边测试。

- 两种输入路径与各自处理
  - 直接 argv 传 heredoc（GPT‑4.1 常见）
    - 形如 ["apply_patch", "&lt;&lt;'EOF'\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]。
    - 处理路径：由 Lenient 模式直接剥离 heredoc，内部继续严格校验补丁体。参见 [check_patch_boundaries_lenient()](apply-patch/src/parser.rs:199)。
  - 通过 shell 解释 heredoc（推荐的 shell 语义做法）
    - 形如 ["bash","-lc","apply_patch &lt;&lt;'EOF'\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]。
    - 处理路径：先从整条 bash 命令中解析出 heredoc 正文，再将正文作为补丁体解析。heredoc 提取逻辑见 [apply-patch/src/lib.rs](apply-patch/src/lib.rs:267) 中基于 tree‑sitter‑bash 的 AST 解析流程，随后仍进入统一的补丁语法解析流程。

- 工具提供策略与文档注入的对应关系
  - 并非所有模型都提供 apply_patch 作为「函数工具」。是否暴露 apply_patch 作为 Tool 由模型家族标记 uses_apply_patch_tool 决定（例如 "gpt‑oss"），参见 [find_family_for_model()](core/src/model_family.rs:68) 中 "gpt‑oss" 分支。配置层会据此决定是否在工具清单中包含 apply_patch（见 [config.rs](core/src/config.rs:670) 对 include_apply_patch_tool 的默认逻辑）。
  - 对 GPT‑4.1，因为其对本地命令/工具的行为差异，选择通过注入「使用说明文档」[`APPLY_PATCH_TOOL_INSTRUCTIONS`](apply-patch/src/lib.rs:23) 来明确告诉模型如何用 shell/CLI 方式调用 apply_patch，从而规避 argv 传 heredoc 与非 shell 语义的不匹配。

- 针对模型的实践建议
  - GPT‑4.1：
    - 遵循注入的工具使用说明（[`apply_patch_tool_instructions.md`](apply-patch/apply_patch_tool_instructions.md)），可以直接按文档示例通过 shell+apply_patch 传递 heredoc。Lenient 解析会兜底处理 GPT‑4.1 偶发的「把 heredoc 当 argv」问题。
    - 如需最大化环境可移植性，优先采用 shell 语义（["bash","-lc","apply_patch &lt;&lt;'EOF' ..."]）形式，确保 heredoc 由 shell 解释。
  - 其他模型：
    - 若模型家族启用了 uses_apply_patch_tool，则以工具调用的方式调用 apply_patch；否则使用 shell 命令形式。无论何种方式，补丁体都必须满足 apply‑patch 语法规范。

小结：
- 只有当模型家族标记 needs_special_apply_patch_instructions=true（目前是 GPT‑4.1）时，系统才会把额外的 apply_patch 使用说明拼接进模型指令中（见 [Prompt::get_full_instructions()](core/src/client_common.rs:44)、[find_family_for_model()](core/src/model_family.rs:68)）。
- 为兼容 GPT‑4.1 常见的 heredoc 作为 argv 的调用方式，解析器默认启用 Lenient（[PARSE_IN_STRICT_MODE](apply-patch/src/parser.rs:47)=false），在不降低补丁体语法校验的前提下，对外层 heredoc 包裹进行温和处理（见 [ParseMode](apply-patch/src/parser.rs:115)、[check_patch_boundaries_lenient()](apply-patch/src/parser.rs:199)）。
