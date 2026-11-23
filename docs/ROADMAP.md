# AutomationKitBundle roadmap

Design contract: [CONTRACT.md](CONTRACT.md) (contract 254-AK v0.1; every Q-AK answered by the owner on 2026-11-23 as recommended). Every slice is
one pull request on `nowo-tech/AutomationKitBundle`, merged with CI green, followed by its tag. The answers are
recorded in CONTRACT.md §22. **Packagist:** the owner registers the package after S0 is on `main`, before the 0.1.0
tag.

Audit rule (2026-11-07 batch audits): S1 (access, tenancy, idempotency) and S5 (mail and webhooks sent by
automations, spec 254 *Immediate audit*) get an **immediate** SEC-004 audit before their tag; the other slices are
audited in the batch before 1.0.0.

## Consumers and order

| Consumer | Needs | Target |
| --- | --- | --- |
| CrmKit F-3 (0.24.0, Beacon v1.184) | S1 – S5: engine, delays, sequences, UI, MailKit mail with `sending_domains` | CrmKit builds its triggers and actions against 0.1.0 (S1) |
| MailKit F-0 (`sending_domains`) | None from this bundle; S5 depends on it | MailKit minor, before S5 |
| Beacon spec 216 task automations | 1.0.0 + TaskBoard bridge | After 1.0 (Q-AK-13) |

## Slices

| Slice | Version | Content | Done when | Audit |
| --- | --- | --- | --- | --- |
| **S0** scaffold | `main`, no tag | Composer, Docker, Makefile + `release-check`, CI matrix (SF 7.4 / 8.0 / 8.1 × MySQL / PostgreSQL / SQLite), PHPStan + `phpstan-frankenphp`, Igor, Rector, CS, coverage gate, Spec Kit baseline, docs set, `.github`, release / sync workflows, Flex recipe, demo skeleton (FrankenPHP worker, MySQL, Messenger Doctrine transport), strict config tree with `enabled: false` | `make release-check` green; scaffold REQ checklist 100 %; **owner registers the package on Packagist** | Batched |
| **S1** engine core | 0.1.0 | Domain VOs, XML + attribute mapping, private EM, DBAL stores in transaction-doctrine write units; catalogue (trigger sources, condition evaluators, action handlers, subject resolvers, param schema); `AutomationEventsInterface` with the after-commit buffer; matching through the trigger index; runs and run steps with unique keys; step executor (gate, action, stop); chain context, cycle check, depth and visited set; tenant model; access checker + author re-check; throttles and counters; global and per-action kill switch; run log; signed messages; `AutomationAdminInterface` (create, update, activate, pause, archive) without UI; console tick | Idempotency under two workers (SC-005); loop stops at depth; worker-mode test; both host kinds | **Immediate** |
| **S2** time | 0.2.0 | `delay` and `wait_until` steps, execution windows (time zone, weekdays, hours, DST), Scheduler recipe (tick, scheduled triggers, retention), `ScheduledTriggerSourceInterface`, `signal()`, leases and expired-lease recovery, backoff retries, circuit breaker, retention job | Window reference table (20 cases); lease recovery test | Batched |
| **S3** sequences | 0.3.0 | Sequence kind, enrolments (unique active key), `SequencesInterface` (enrol, bulk enrol, unenrol, stop for subject), stop rules by signal, re-enrolment policy, daily cap per sender, `enrol` / `unenrol` steps, sequence events and triggers | Double-enrol race test; stop signal racing a step test | Batched |
| **S4** UI | 0.4.0 | UiKit + FormKit pages: automations list, editor with sentence preview, step and condition dialogs on demand (remote form dialog, no-JS links), activation and dry-run dialogs (`simulate()`), run log + run detail, enrolments, operator page (kill switches, backlog); Stimulus controllers; translations ×7; embeddable list per context; page contracts; e2e with screenshots | Page contract, axe, query budgets, Playwright e2e per permission | Batched |
| **S5** actions and bridges | 0.5.0 | MailKit bridge (`mail.send`, engine-level sending-domain guard, activation check, daily domain re-check, consent and unsubscribe ports), NotifyKit (`notify`, author notices), TaskBoard (spec 216 triggers and actions), Workflow (`workflow.apply_transition`, `workflow.transition_applied`), OutboundUrlGuard (`http.webhook`, off by default), WorkingTime holidays | Each bridge tested with the real bundle in require-dev; mail refusal paths; webhook SSRF table | **Immediate** (mail, webhooks) |
| **S6** GDPR, audit, API | 0.6.0 | `AutomationSubjectsInterface` (forget subject / tenant / context, export), UserKit eraser + export, AuditKit bridge, InstanceSettings runtime settings, `Application/Api/Operation/*`, optional public API `<prefix>/automations/v1` + OpenAPI, ApiToken module `automations`, docs (USAGE, CONFIGURATION, CONCURRENCY, TEMPLATES, FRANKENPHP-WORKER-AUDIT, UPGRADING) | Export ↔ erasure parity; OpenAPI route test | **Immediate** for the API; batched for the rest |
| **S7** 1.0 | **1.0.0** | `bundle-req-audit` 100 %, SEC-004 batch audit over S0 – S6, demo polish, Beacon integration notes | REQ audit 100 %, audits pass, owner OK | Batched |
| 1.x | 1.1+ | SmsKit `sms.send` (Q-AK-11), definition create / edit through the API (Q-AK-12), if / else branches if asked (Q-AK-2), Beacon spec 216 migration guide (Q-AK-13) | — | Per minor |

## Decisions

Q-AK-1 … Q-AK-17: decided 2026-11-23, all as recommended ([CONTRACT.md §22](CONTRACT.md#22-answers)).
