# Lens: AI / LLM Integration

LLM features bolted onto a Rails app — a chat endpoint, a "summarize this", a RAG search, an agent with tools. Two arrival paths, same risks: a vibecoded app that shipped with an LLM from day one, and a legacy app that wired a model into code never designed for untrusted input or outbound calls to a third party. The engines don't model this at all — Brakeman can't tell a string came from a model, and no scanner reads a prompt. Read `app/services/`, `app/jobs/`, `lib/`, the controllers behind chat/completion endpoints, and any view that renders model output — anywhere `OpenAI`, `Anthropic`, `RubyLLM`, `Langchain`, `raix`, or a raw `Faraday`/`Net::HTTP` call to `api.openai.com`/`api.anthropic.com` appears.

Reference: OWASP LLM Top 10 2025 and the OWASP LLM Prompt Injection Prevention Cheat Sheet — link findings to them.

## Prompt injection (LLM01)

- **User input concatenated straight into the prompt** — `"Summarize this: #{params[:text]}"` lets the user rewrite the system's intent. The finding is not the interpolation itself but what the model can *do* with the hijacked instruction — chase it to the downstream effect (output rendered as HTML, a tool call, a data read).
- **Indirect injection through retrieved content** — RAG documents, scraped pages, email bodies, prior chat turns, and DB rows fed to the model are *untrusted input*, not data. A poisoned document steers the model exactly like a malicious user can. Most teams trust retrieved content implicitly; that's the gap.
- Remedy: keep instructions and data in separate roles (system vs. user), never build the system prompt from user/retrieved strings, and treat every model action as attacker-influenced for the checks below.

## Improper output handling (LLM05) — the highest-yield check here

- **Model output rendered with `raw` / `html_safe` / `<%==`** — `raw(@completion)` or `markdown(@answer).html_safe` is stored XSS: the model emits `<script>` because a user told it to, and the view trusts it. Brakeman won't flag this — it can't tell the string originated from an LLM. This is the confirmable finding in this lens; name the view and the sink.
- **Model output rendered as markdown without sanitization** — Redcarpet without `::Safe`, CommonMarker without `:SAFE` (pairs with `arsenal/hardening.md`).
- **Model output piped into a dangerous sink** — `eval`, `send`, `constantize`, `system`/backticks, raw SQL, or `redirect_to(model_output)` (open redirect). An "agent" that writes code or SQL and then runs it is the worst case.
- Remedy: treat model output exactly like raw user input — escape by default, sanitize markdown with the safe renderer, and never feed it to a code/SQL/command sink without validation.

## Sensitive data disclosure (LLM02)

- **PII or secrets sent into the prompt** — whole user records, documents, transcripts, or keys shipped to a third-party model API with no redaction. The data leaves your trust boundary and may be logged or retained by the provider. Name the data class (health, financial, credentials) — it sets the severity.
- **Secrets embedded in the system prompt** — API keys, internal URLs, or another service's tokens baked into the prompt text leak on the first successful injection.
- Pairs with `arsenal/logging.md` (prompts and responses logged verbatim with PII) and `arsenal/secrets.md` (the provider API key itself hardcoded instead of in credentials/ENV).

## Excessive agency / tool-use (LLM06)

When the model has tools/function-calling, the injection target stops being text and becomes *actions*.

- **A tool that fetches a URL the model chooses** — server-side request forgery: the model, steered by injection, hits internal services, cloud metadata (`169.254.169.254`), or `file://`. Remedy: an allowlist of hosts and `ssrf_filter` on the fetch — never a denylist.
- **Tools with DB, shell, mailer, or filesystem reach** — broad capability with no scoping means an injected instruction acts with the app's full privileges. Remedy: the smallest possible tool surface, per-tool authorization, and the same ownership scoping `arsenal/authorization.md` demands.
- **No human-in-the-loop on irreversible actions** — sending money, deleting records, or emailing customers driven directly by model output. Remedy: a confirmation gate on high-impact tools.

## Unbounded consumption (LLM10)

- **No rate or cost ceiling on an LLM-backed endpoint** — every request costs real money, so an unthrottled endpoint is both a DoS and a wallet-drain. Remedy: Rails 8's `rate_limit` or rack-attack, plus a per-user and total token/spend cap (pairs with `arsenal/api.md`).

## Severity and remedies

Model output rendered as HTML (`raw`/`html_safe`) and an SSRF-capable or destructive tool surface are **confirmed-critical** when you can show the path — state who injects what and where it lands (a `<script>` in a shared chat log; an internal URL fetched by a tool). Unredacted PII or secrets to a third-party API rank with the hardening tier, higher by data class. **Prompt injection on its own is "theoretical" until you articulate the downstream effect** — XSS, data exfiltration, or an unauthorized action; honor the project's adversarial-verification bar and never sell a bare "the prompt is injectable" as confirmed. Remedies name the move: escape/sanitize output, split instruction and data roles, redact before the API call, allowlist + `ssrf_filter` on model-driven fetches, minimal scoped tools with confirmation gates, and `rate_limit`/rack-attack with a spend cap.
