# Architectural Diagnostic Sample

This repository contains an anonymized sample architectural report generated during a large-scale calibration run on a major Go codebase (~2,500 files).

The goal of the project is to analyze software systems from a structural and architectural perspective, with a strong focus on:

architectural consistency
hidden coupling
boundary violations
dependency direction
aggregate integrity
long-term maintainability

The current engine is intentionally conservative and non-commercial.
The objective is not to generate “AI noise”, but to surface high-signal architectural issues with a low false-positive rate.

This sample report is provided to demonstrate:

analysis depth
reporting style
architectural reasoning quality
signal/noise ratio
Calibration Phase

I am currently running a limited free calibration phase on real-world repositories in order to refine:

false-positive detection
severity scoring
architectural relevance
report clarity

Because deep analysis runs have a non-trivial compute cost, only a limited number of repositories can be processed during this phase.

## Interested?

If you'd like your repository analyzed, feel free to reach out.

LinkedIn:
https://www.linkedin.com/in/hugo-vantighem-9669a14b/

You can also open an issue or contact me directly.

> **Sample excerpt.** This is a representative extract of a First-Pass Architectural Diagnostic: the findings overview table and one fully detailed finding. The target codebase has been anonymized. Wording, structure and depth are identical to a real deliverable.

# First-Pass Architectural Diagnostic — [Anonymized Go Platform]

**Target:** [anonymized] · **Branch/commit:** [redacted] · **Date:** [redacted]
**Scope:** security · reliability · concurrency · architecture · **Method:** multi-agent audit + cross-falsification (read-only)

---

## 1. Overall verdict

**WARN**

- **What holds:** Internal IPC authentication correctly constrained (Unix socket / loopback + HMAC token), queue system graceful shutdown working, migrations isolated with TCP `DialContext` protection, optional TLS consistent with documented use cases.
- **To watch:** Silent webhook loss on crash (fire-and-forget design with no requeue), direct `models/ → modules/setting` coupling degrading testability across 70+ files, queue goroutine with no safety net against panics.
- **Fix immediately:** The webhook HTTP transport ignores `allowedHostMatcher` on the default configuration (`ProxyURL = ""`) and implements no `DialContext` — any cloud-deployed instance is exposed to SSRF against metadata endpoints (169.254.169.254, 100.64.0.0/10).

---

## 2. Findings table (substantive only: FAIL · WARN)

| # | Status | Severity | Component | Title | File:line |
|---|--------|----------|-----------|-------|-----------|
| S-1 | FAIL | HIGH | Webhook delivery | SSRF: `allowedHostMatcher` ignored on default config, no TCP `DialContext` | `services/webhook/deliver.go:283-327` |
| REL-7 | WARN | MEDIUM | Webhook delivery | Silent loss: `handler()` always returns `nil`, `MarkTaskDelivered` precedes HTTP send | `services/webhook/webhook.go:103` · `deliver.go:207` |
| ARC-7 | WARN | MEDIUM | Models | Direct `models/ → modules/setting` coupling (70+ files): degraded testability, cascading recompiles | `models/user/user.go:267,277,282,383` · `models/perm/access/repo_permission.go:202` |
| REL-3b | WARN | LOW | Queue | `popItemByChan`: goroutine with no `defer recover()`, panic = process crash | `modules/queue/base.go:25-40` |
| S-4 | WARN | LOW | Internal IPC | `InsecureSkipVerify: true` hardcoded on IPC transport | `modules/private/internal.go:59-63` |

---

## 3. Detailed finding (sample — 1 of 5)

### S-1 — SSRF: `allowedHostMatcher` bypass + missing TCP protection

**FAIL** · **Severity: HIGH** · **Confidence: HIGH** (source code verified)

**Evidence:** `services/webhook/deliver.go:283-309`

```go
func webhookProxy(allowList *hostmatcher.HostMatchList) func(req *http.Request) (*url.URL, error) {
    if setting.Webhook.ProxyURL == "" {
        return proxy.Proxy()   // ← allowList ignored — early return
    }
    // ... the allowList check is only reached when ProxyURL != ""
}
```

`proxy.Proxy()` (verified at `modules/proxy/proxy.go:56-63`) returns `http.ProxyFromEnvironment` when `Proxy.Enabled == false` or `ProxyURL == ""` — with no host filtering whatsoever. The `webhookHTTPClient` (lines 321-327) only declares `Proxy: webhookProxy(allowedHostMatcher)`; there is no `DialContext`. Contrast with the correct pattern in `services/migrations/http_client.go:27`:

```go
// migrations — correct pattern:
DialContext: hostmatcher.NewDialContext("migration", allowList, blockList, setting.Proxy.ProxyURLFixed),

// webhook — missing:
// no DialContext in &http.Transport{...}
```

**Reproduction scenario:** Instance deployed on AWS/GCP/Azure with default config (`WEBHOOK_PROXY_URL` empty, `ALLOWED_HOST_LIST` unset) → a contributor account creates a repo → adds a webhook pointing to `http://169.254.169.254/latest/meta-data/iam/security-credentials/` → triggers a push → `webhookProxy` returns `proxy.Proxy()` with no filtering → the HTTP request is sent to the metadata service → IAM credentials appear in the webhook delivery history in the UI.

**Counter-evidence considered:** `ALLOWED_HOST_LIST` can be explicitly configured to restrict allowed hosts — but that does not change the `ProxyURL == ""` path that bypasses the list. The default value is `MatchBuiltinExternal` (line 317), which excludes RFC1918 ranges — but that value never reaches `DialContext`, only the `Proxy` func, which is short-circuited. An operator who configures `ProxyURL` benefits from filtering; the default configuration does not use it.

**Recommendation:** Add `DialContext: hostmatcher.NewDialContext("webhook", allowedHostMatcher, nil, setting.Proxy.ProxyURLFixed)` to the `&http.Transport{}` of `webhookHTTPClient` (mirroring `migrations/http_client.go:27`). Simultaneously fix `webhookProxy` to propagate `allowList` in the `ProxyURL == ""` branch.

