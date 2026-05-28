---
name: openclash-ai-rule-provider
description: Add or maintain the OpenClash_Overwrite repository's dedicated AI service rule-provider. Use when Codex needs to route a new AI service domain such as jetbrains.ai, claude.ai, perplexity.ai, or similar through the AI服务 proxy group by editing Rules/Myself_AI_Domain.list and Yaml/Overwrite-Clash.yaml.
---

# OpenClash AI Rule Provider

## Workflow

1. Confirm the repository root contains `Yaml/Overwrite-Clash.yaml`.
2. Add AI service domains to `Rules/Myself_AI_Domain.list` using classical Clash rule syntax.
3. Ensure `Yaml/Overwrite-Clash.yaml` has this rule before the generic `category-ai` rule:

```yaml
  - RULE-SET,Myself_AI_Domain,AI服务
  - RULE-SET,category-ai,AI服务
```

4. Ensure `rule-providers` contains this provider:

```yaml
  Myself_AI_Domain:                         {<<: *class,          url: "https://raw.githubusercontent.com/wangqiankun/OpenClash_Overwrite/refs/heads/main/Rules/Myself_AI_Domain.list"}
```

5. Keep changes minimal. Do not reformat unrelated YAML lines.

## Rule Format

Use `DOMAIN-SUFFIX` for a domain and all subdomains:

```text
DOMAIN-SUFFIX,jetbrains.ai
```

Use `DOMAIN` only when exactly one hostname should match:

```text
DOMAIN,example.jetbrains.ai
```

Use `DOMAIN-KEYWORD` only when a broad keyword match is intentional.

## Verification

After edits, run:

```bash
ruby -e 'require "yaml"; YAML.load_file("Yaml/Overwrite-Clash.yaml", aliases: true)'
git diff --check
```

If the skill itself was changed, also run:

```bash
python3 /Users/wangqiankun/.codex/skills/.system/skill-creator/scripts/quick_validate.py .codex/skills/openclash-ai-rule-provider
```
