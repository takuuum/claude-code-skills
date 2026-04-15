---
description: Code Review with Codex MCP and everything-claude-code:code-review, running as mutual cross-review by default. Invoke when the user wants to review code, check a PR, or audit staged changes.
---

# Code Review with Codex MCP

Codex MCP と Claude（everything-claude-code:code-review）を並列実行し、互いの結果を批評し合う**相互レビュー**をデフォルトで行います。単独レビューが必要な場合は `--no-cross-review` で無効化できます。

## Arguments

`$ARGUMENTS` には以下を指定できます（省略可）:
- ファイルパス: `src/components/Button.tsx` など（複数指定可）
- `--diff` / `-d`: ステージング済みの差分をレビュー
- `--pr <number>` / `-p <number>`: 指定した PR の差分をレビュー
- `--focus <area>`: レビューの焦点（security / performance / style / logic）
- `--no-cross-review`: 相互レビューを無効にして Codex 単独レビューに切り替える

引数なしの場合は、ステージング済みの差分 → なければ `HEAD~1..HEAD` の差分をレビュー対象にします。

---

## Step 1: レビュー対象の取得

`$ARGUMENTS` を解析して、以下のいずれかでコードを取得します。

- ファイルパス指定あり → 該当ファイルの内容を読み込む
- `--diff` or 引数なし → `git diff --staged`（空なら `git diff HEAD~1 HEAD`）
- `--pr <number>` → `gh pr diff <number>`

取得したコード・差分を変数として保持します。

---

## Step 2: コンテキストの収集

以下を並列で取得します:
- `git log --oneline -5` — 直近のコミット履歴
- `git diff --stat HEAD~1 HEAD` — 変更ファイルのサマリ（PR指定時は `gh pr view <number> --json files`）
- プロジェクトの主要言語・フレームワークの特定（package.json / go.mod / Cargo.toml / pyproject.toml など）
- `.github/claude-rules/coding-standards*.md` が存在する場合は読み込む — プロジェクト固有のレビュー基準として、Step 3 の全エージェントプロンプトに「Project Review Rules」セクションとして含める

---

## Step 3: レビュー実行

`--no-cross-review` が指定されている場合のみ単独モードで実行します。それ以外はデフォルトで相互レビューモードを使用します。

### 単独モード（`--no-cross-review` あり）

`codex` MCP ツールを使って、以下のプロンプトでレビューを依頼します:

```
You are a senior software engineer conducting a thorough code review.

## Target Code
[Step 1 で取得したコード・差分を挿入]

## Project Context
- Language/Framework: [Step 2 で取得した情報]
- Recent commits: [git log の結果]
- Project-specific review rules: [Step 2 で取得した .github/claude-rules/coding-standards*.md の内容（存在する場合）]

## Review Instructions
Analyze the code and provide feedback in the following categories:

### 1. Bugs & Correctness
- Logic errors, off-by-one errors, incorrect assumptions
- Unhandled edge cases and null/undefined risks
- Race conditions or concurrency issues

### 2. Security
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication/authorization gaps
- Exposed secrets or sensitive data
- Input validation issues

### 3. Performance
- Inefficient algorithms or data structures (note Big-O if relevant)
- Unnecessary re-renders or recomputations
- N+1 query problems or missing indexes
- Memory leaks

### 4. Code Quality
- Readability and clarity
- Naming conventions and consistency
- DRY violations and unnecessary complexity
- Missing or misleading comments

### 5. Best Practices
- Framework/language-specific conventions
- Design pattern adherence
- Test coverage considerations

For each issue, provide:
- **Severity**: Critical / Warning / Suggestion
- **Location**: File and line reference if possible
- **Description**: What the problem is
- **Recommendation**: How to fix or improve it

Also note any **positive highlights** worth acknowledging.
```

`sandbox: "read-only"` で実行します（ファイル読み取り専用、変更なし）。

**Codex MCP 接続失敗時のリトライ:**
`codex` MCP ツールの呼び出しが接続エラー・認証エラーで失敗した場合:
1. `codex login` を実行して再認証する
2. 成功後、同じプロンプトで `codex` MCP ツールを再度呼び出す
3. 再試行も失敗した場合はエラー内容をユーザーに報告して処理を中断する

Step 4 に進みます。

---

### 相互レビューモード（デフォルト）

2 つのレビューを **並列 Task エージェント** で同時実行します。

#### Agent A — Claude スタイルレビュー（Claude 直接）

Step 2 で `.github/claude-rules/coding-standards*.md` を取得済みの場合、以下の標準観点に**加えて**、そのルールを「Project-Specific Rules」として最優先で適用します。

以下の観点でコードを分析します:

**Security Issues (CRITICAL):**
- Hardcoded credentials, API keys, tokens
- SQL injection, XSS, command injection vulnerabilities
- Missing input validation
- Insecure dependencies / path traversal risks

**Code Quality (HIGH):**
- Functions > 50 lines
- Files > 800 lines
- Nesting depth > 4 levels
- Missing error handling
- console.log / debug statements left in code
- TODO/FIXME comments
- Missing JSDoc for public APIs
- Mutation patterns (use immutable instead)

**Best Practices (MEDIUM):**
- Missing tests for new code
- Accessibility issues (a11y)

各 Issue を `{severity, file, line, description, suggestion}` 形式で返します。

#### Agent B — Codex MCP レビュー

`codex` MCP ツールを `sandbox: "read-only"` で呼び出し、Bugs & Correctness / Security / Performance / Code Quality / Best Practices を分析します（単独モードと同じプロンプト）。Step 2 で取得したプロジェクト固有ルール（`.github/claude-rules/coding-standards*.md`）がある場合は、Codex に送るプロンプト内に「## Project-Specific Review Rules」セクションとして含めます。

**Codex MCP 接続失敗時のリトライ:**
`codex` MCP ツールの呼び出しが接続エラー・認証エラーで失敗した場合:
1. `codex login` を実行して再認証する
2. 成功後、同じプロンプトで `codex` MCP ツールを再度呼び出す
3. 再試行も失敗した場合は Agent B の結果を空としてレポートし、Claude（Agent A）単独の結果で処理を続行する

各 Issue を `{severity, file, line, description, recommendation}` 形式で返します。

---

#### Phase 2: 相互批評（クロスレビュー）

Agent A・B の結果が揃ったら、2 つの **並列批評エージェント** を起動します。

**批評エージェント X — Claude が Codex を批評:**

Agent B（Codex）の Issue リストを受け取り、以下を評価します:
- 誤検知（False Positive）か？ → 根拠を示して反論
- 実際にはより深刻か？ → 深刻度の上方修正を提案
- 見落とし（False Negative）はあるか？ → Agent A が指摘して Codex が見逃した Issue を追記

**批評エージェント Y — Codex が Claude を批評:**

Agent A（Claude）の Issue リストを受け取り、以下を評価します:
- 誤検知（False Positive）か？ → 根拠を示して反論
- 実際にはより深刻か？ → 深刻度の上方修正を提案
- 見落とし（False Negative）はあるか？ → Agent B が指摘して Claude が見逃した Issue を追記

---

#### Phase 3: 統合レポート生成

4 エージェントの出力（A・B・X・Y）を統合して最終レポートを生成します:

```
## Cross-Review Report

### Consensus Issues（両者が合意した問題）
両 Reviewer が指摘した Issue。信頼度が高い。

### Claude-Only Issues（Claude のみが指摘）
Codex が見逃し、かつ批評 Y で反論されなかった Issue。

### Codex-Only Issues（Codex のみが指摘）
Claude が見逃し、かつ批評 X で反論されなかった Issue。

### Disputed Issues（意見が割れた問題）
一方が指摘し、他方が False Positive と反論した Issue。双方の根拠を併記。

### Cross-Critique Summary
- Claude → Codex: [批評 X のサマリ]
- Codex → Claude: [批評 Y のサマリ]
```

Step 4 に進みます（通常モードの出力形式に沿って整形）。

---

## Step 4: レビュー結果の出力

Codex の回答（通常モード）または統合レポート（相互レビューモード）を以下の形式に整形して表示します:

```
## Code Review Results

### Overview
[全体的な評価・サマリ]

---

### Issues Found

#### 🔴 Critical
[クリティカルな問題]

#### 🟡 Warnings
[警告レベルの問題]

#### 🔵 Suggestions
[改善提案]

---

### ✅ Positive Highlights
[良い点]

---

### Next Steps
[優先して対応すべきアクション]
```

相互レビューモードでは `Issues Found` の前に `### Consensus / Claude-Only / Codex-Only / Disputed` セクションを挿入します。

---

## Notes
- `--focus security` のように焦点を指定した場合、そのカテゴリを重点的にレビュー
- レビュー対象が大きい場合は、ファイルごとに分割して Codex を呼び出す
- 相互レビューはデフォルト動作。並列 Task エージェントを 4 つ起動するため、単独モードより時間がかかる
- 速度優先の場合は `--no-cross-review` で Codex 単独レビューに切り替え可能
- 日本語でのフィードバックが必要な場合は `$ARGUMENTS` に `--lang ja` を追加
