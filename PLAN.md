## Solution plan

**Issue:** [Prompt injection defense doesn't sanitize newline characters in user-supplied resume text (#64)](https://github.com/ascherj/pathreview/issues/64)

### Understand
In the `PromptDefense` class (`safety/prompt_defense.py`), `sanitize()` does not actually strip the newline-based injection patterns that `is_injection_attempt()` detects (e.g. `\n---\n` separators, fake `\nSystem:`/`\nHuman:`/`\nAssistant:` role switches, "Ignore/Forget/Disregard/Override" directives) — it only removes angle brackets and template delimiters (`{{`, `}}`, `{%`, `%}`). Reproduced in `tests/unit/test_prompt_defense.py::test_sanitize_removes_newline_injection_patterns` (commit `1f5c8ae`): a malicious payload survives `sanitize()` untouched and still trips `is_injection_attempt()` afterward. We also found a deeper issue: neither `sanitize()` nor `is_injection_attempt()` is called anywhere in production code today — only in their own unit tests — so the defense is fully inert in the real resume ingestion path.

### Map
- `safety/prompt_defense.py` — fix `sanitize()` to neutralize `INJECTION_PATTERNS` matches, not just template/angle-bracket characters
- `ingestion/pipeline.py` (`IngestionPipeline.ingest_resume`) — wire in a call to `PromptDefense.sanitize()`/`is_injection_attempt()` on parsed resume text before chunking
- `ingestion/parsers/resume_parser.py` — confirm where raw resume text first becomes available as a string, to decide the exact sanitize call site
- `tests/unit/test_prompt_defense.py` — extend/adjust unit tests for the fixed `sanitize()`
- Ingestion tests (e.g. `tests/unit/test_pipeline.py` or similar, if one exists) — add a test confirming a malicious resume is sanitized before chunking

### Plan
1. Extend `sanitize()` in `safety/prompt_defense.py` to strip/neutralize each pattern in `INJECTION_PATTERNS` (not just the four hardcoded characters), reusing the existing `INJECTION_PATTERNS` list instead of duplicating regexes.
2. Call `PromptDefense.sanitize()` on parsed resume text inside `IngestionPipeline.ingest_resume()` (`ingestion/pipeline.py`), before the text is passed to `strategy_selector.chunk()`.
3. Call `PromptDefense.is_injection_attempt()` on the raw text first (before sanitizing) so an attempt is logged/flagged, then sanitize regardless of the result.
4. Update/add unit tests in `tests/unit/test_prompt_defense.py` covering the fixed `sanitize()` behavior for each injection pattern.
5. Add an ingestion-level test confirming a malicious resume passed through `ingest_resume()` ends up sanitized before chunking/embedding.

### Inputs & outputs
Input: raw user-supplied resume text (`str`, from `ResumeParser.parse()`). Output: sanitized text with injection patterns stripped/neutralized, safe to chunk, embed, and later include in LLM prompts — with the original detection behavior (`is_injection_attempt`) preserved for logging/observability.

### Risks & unknowns
- Over-aggressive stripping could mangle legitimate resume content that happens to contain `---` (e.g. a markdown section divider) or the word "System" (e.g. "Operating Systems" experience) — need to avoid false-positive content loss.
- Unclear whether `is_injection_attempt()` should just log a warning (current behavior) or actively reject/block ingestion of a resume outright once sanitized — needs a product decision.
- Unsure if there are other entry points besides `ingest_resume()` (e.g. README or repo metadata ingestion) that should also route through `PromptDefense`.

### Edge cases
- Legitimate resumes using `---` as a section divider
- Resumes mentioning "System", "Human", or "Assistant" as part of job titles/skills (e.g. "Systems Engineer")
- Empty or whitespace-only text
- Text sanitized twice (idempotency — mirrors existing `test_sanitize_idempotent`-style coverage)
- Multiple distinct injection patterns present in a single resume at once