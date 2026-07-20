## Week 7 — Issue selection

**Issue link:** [#64](https://github.com/ascherj/pathreview/issues/64)

**Issue title:** Prompt injection defense doesn't sanitize newline characters in user-supplied resume text

**Tier:** [ ] Tier 1  [X] Tier 2  [ ] Tier 3

**Problem summary:**
The `PromptDefense` class in `safety/prompt_defense.py` detects newline-based prompt injection patterns (e.g., `\n---\n` separators, `\nSystem:` role switches, "Ignore/Forget/Disregard" directives) via `is_injection_attempt()`, but its `sanitize()` method never actually strips or neutralizes them — it only removes angle brackets and template delimiters (`{{`, `}}`, `{%`, `%}`). This means a resume submitted with embedded newlines and fake role markers can pass through sanitization untouched and break out of the system prompt to inject unauthorized instructions into the LLM context. The vulnerability affects the resume-review pipeline anywhere user-supplied resume text is sanitized before being included in a prompt. A successful fix would extend `sanitize()` to actually remove or escape the newline-triggered injection patterns already defined in `INJECTION_PATTERNS`, so detected attack strings are neutralized rather than just logged/flagged.

**Branch name:** safety/64-newline-prompt-injection-sanitation-defense

**Setup confirmation:** [X] App runs locally at localhost:5173

**Cohort ledger:** [N/A] Issue added to cohort ledger