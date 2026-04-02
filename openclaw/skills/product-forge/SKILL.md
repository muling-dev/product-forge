---
name: product-forge
description: >
  Autonomous product builder. Receives a requirement via Telegram or any channel,
  spawns a Claude Code session to build the product, and pushes the result back.
  Trigger: "forge <requirement>", "/forge <requirement>", or any message asking to build/create a product.
metadata:
  openclaw:
    emoji: "⚙️"
    requires:
      anyBins: ["claude"]
---

# product-forge (OpenClaw trigger)

When triggered with a product requirement:

## Step 1: Confirm receipt
Reply immediately:
```
⚙️ [product-forge] 收到需求，启动中...
```

## Step 2: Spawn Claude Code via ACP

Use `sessions_spawn` with:
- `runtime: "acp"`
- `agentId: "claude"`
- `cwd: "~"`
- `streamTo: "parent"`
- `task`: the full requirement text, prefixed with `/forge `

Example task string:
```
/forge <the user's exact requirement text>
```

## Step 3: Listen for completion signal

Wait for an `openclaw system event` with text starting with:
- `forge-done:` → success
- `forge-error:` → failure
- `forge-retry-*:` → sprint retry (send brief update to user)
- `forge-phase1-done:` → phase 1 complete (send brief update to user)

## Step 4: Push result to user

**On `forge-done:`:**
Parse the event text: `forge-done: <name> | <type> | <url> | <dir>`

Reply to the user's original message:
```
✅ [product-forge] 完成交付

项目: <name>
类型: <type>
目录: <dir>
[if url] 预览: <url>

规格文档: <dir>/.product-forge/spec.md
```
Attach any screenshots from `<dir>/.product-forge/screenshots/`.

**On `forge-error:`:**
Reply: `❌ [product-forge] 出错: <error message>`

**On `forge-retry-*:`:**
Reply: `🔄 [product-forge] Sprint 评估未通过，自动重试中...`

**On `forge-phase1-done:`:**
Reply: `📋 [product-forge] 需求已理解，开始规划...`
