# Apply-Patch 机制详解

本文档解释 Codex CLI 中用于安全、稳定地修改文件的 apply-patch 机制，并结合最新实现细节进行了全面更新与校准。

## 概述

Apply-patch 是 Codex CLI 的核心文件编辑机制，它采用“上下文匹配”而非“行号定位”的策略来应用补丁。相较传统行号 diff，这种方式在自动化场景下更稳健，能更好地抵抗并行修改、格式化以及轻微的非语义变更。

## 语法规范

### 补丁包围结构

每个补丁必须以以下标记包围：

```bash
*** Begin Patch
[一个或多个文件操作]
*** End Patch
```

### 文件操作类型

1) 添加文件 (Add File)
```bash
*** Add File: path/to/file.py
+第一行内容
+第二行内容
```

2) 删除文件 (Delete File)
```bash
*** Delete File: path/to/obsolete.py
```

3) 更新文件 (Update File，可选重命名)
```bash
*** Update File: path/to/file.py
*** Move to: path/to/new_name.py   # 可选
[一个或多个修改块]
```

### 修改块语法

- 修改块以 @@ 开头，后面可选一个“单行上下文头”（通常是类名、函数名等）。
- 修改块正文中的每一行必须以下列前缀之一开头：
  - 空格 " " 表示上下文行（原样保留）
  - "-" 表示要删除的旧行
  - "+" 表示要添加的新行

基本示例
```bash
@@ def greet():
 print("Hello")
-return 0
+return 1
```

文件末尾追加示例
```bash
@@
+在文件末尾追加的新行
*** End of File
```

重要说明
- 同一“修改块”内只支持一个 @@ 头；不要在同一块内写多个连续的 @@。如需多级定位或多处修改，请使用多块。
- 一个 Update File 下可以包含多个修改块；从第二个修改块开始，必须显式以 @@ 头起始。
- 首个修改块可省略 @@ 头：当首块没有 @@ 行时，解析器会将紧随其后的带前缀的 diff 行视为该块的正文。

省略首块 @@ 的示例（受支持）
```bash
*** Begin Patch
*** Update File: example.py
 import os
+import sys
*** End Patch
```

多块修改示例（推荐处理多处位置）
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

多级定位（推荐写法）
- 若需要“类 → 方法”这类分层定位：把外层上下文写到 @@ 头里，把更具体的内层锚点写为“上下文行”（以空格开头）置于修改区域之上。
```bash
*** Begin Patch
*** Update File: app/models.py
@@ class User:
 def get_name():
-    return self.name
+    return self.name.title()
*** End Patch
```
说明：这里 @@ 只出现一次（class User），方法名作为上下文行参与匹配；不要在同一块中写两行 @@。

## 上下文匹配机制

应用流程
1) 解析 @@ 后的“单行上下文头”（如有），用于粗定位。
2) 对修改块正文的旧行序列执行模式搜索以精确定位并替换。
3) 当标记了“文件结尾”时，优先从文件结尾尝试匹配。

seek_sequence 算法能力
- 顺序匹配：在文件中搜索一段连续的行序列
- 宽松空白匹配：依次尝试精确匹配、忽略行尾空白、忽略行首尾空白
- Unicode 归一化：常见破折号/引号/不换行空格等会被归一到 ASCII 以提升容错
- EOF 友好：在“End of File”场景优先从文件尾部匹配

端到端示例
```bash
*** Begin Patch
*** Update File: example.py
@@
-print("Hello")
+print("Hello, World!")
*** End Patch
```

## 与 heredoc 的协同与兼容性

Apply-patch 既支持通过 shell heredoc 传入补丁，也兼容某些模型将 heredoc 作为“参数字符串”直接传给可执行文件的做法。

推荐（Shell 解释 heredoc）
```bash
bash -lc 'apply_patch <<EOF
*** Begin Patch
*** Add File: hello.py
+print("hello")
*** End Patch
EOF'
```

兼容（模型将 heredoc 当作参数传入）
```bash
["apply_patch", "<<'EOF'\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]
["applypatch", "<<\"EOF\"\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]
```
- 解析器会在 Lenient 模式下自动剥离 heredoc 包装，仅对补丁体执行严格语法校验。

另一条路径：显式通过 Shell 执行
```bash
["bash","-lc","apply_patch <<'EOF'\n*** Begin Patch\n...\n*** End Patch\nEOF\n"]
```
- 在此路径中，通过内置的 Bash 语法树解析提取 heredoc 正文后统一进入补丁解析流程。

别名
- 命令同时支持别名 applypatch，与 apply_patch 等价。

## 错误处理与调试

错误类型
- ParseError：语法解析错误（包含行号）
- ComputeReplacements：上下文匹配失败（通常是旧行序列或上下文不唯一）
- IoError：文件系统错误（读写/创建/删除等）

常见排查步骤
1) 使用较具体且唯一的上下文（类名/函数签名）作为 @@ 头
2) 确认每一行都带有正确的前缀：" " / "-" / "+"
3) 若在文件尾部追加内容，请添加 "*** End of File"
4) 先尝试较小的独立修改块，避免在一个块中包含相距过远的改动

预览与验证
- 可在上层工具链中使用“dry-run”或“预览 diff”能力（如 unified diff 计算）先查看效果。

## 输出与原子性

成功输出
```text
Success. Updated the following files:
A path/added.txt
M path/modified.txt
D path/deleted.txt
```

原子性与文件操作保障
- 添加文件：先创建目录再写入
- 更新文件：读 → 计算替换 → 写（可选重命名为 Move to）
- 删除文件：直接删除（不可逆）

## 最佳实践

- 优先使用相对路径，避免跨目录的副作用
- 将相关改动放入同一 Update File 下的多个修改块，保持每块聚焦且可定位
- 用明确的 @@ 上下文头提升唯一性；不要在同一修改块内写多个 @@
- 需要“类 → 方法”等多级定位时，将外层上下文放入 @@，更细粒度锚点作为“上下文行”

## 实现参考

主要实现文件与关键位置（供深入理解）
- 解析与语法校验：apply-patch/src/parser.rs
  - ParseMode（Strict/Lenient）与 heredoc 剥离逻辑
  - 单块只支持一个 @@ 头的约束
- 命令解析与 heredoc 抽取：apply-patch/src/lib.rs
  - maybe_parse_apply_patch / maybe_parse_apply_patch_verified
  - extract_heredoc_body_from_apply_patch_command 基于 Tree-sitter Bash
- 上下文匹配算法：apply-patch/src/seek_sequence.rs（空白容忍与 Unicode 归一化）
- 结果汇总输出：apply-patch/src/lib.rs 的 print_summary

## 参考资源

- apply-patch/src/lib.rs
- apply-patch/src/parser.rs
- apply-patch/src/seek_sequence.rs
- apply-patch/apply_patch_tool_instructions.md

---

本文档最后更新于: 2025-08-24
