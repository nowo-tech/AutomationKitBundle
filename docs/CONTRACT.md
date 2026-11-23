# Contract 254-AK: generic automation engine, rules and sequences (AutomationKitBundle). Contract v0.1

Status 2026-11-23: **v0.1, accepted.** The owner answered every Q-AK question on 2026-11-23 with its recommendation
(§21, §22). No code and no tag yet. The repository `nowo-tech/AutomationKitBundle` exists (public, MIT) with an
initial README only. Code follows this contract slice by slice ([ROADMAP.md](ROADMAP.md)), starting with S0
(scaffold). **Packagist:** the owner registers `nowo-tech/automation-kit-bundle` on Packagist after S0 is on `main`,
before the first tag (0.1.0 after S1, Q-AK-17).

- **Approval.** Beacon spec 254 *CRM HubSpot parity*, owner decision **D-20261123-11**, question **Q-254-7** (2026-11-23):
  create `nowo-tech/AutomationKitBundle` (Composer `nowo-tech/automation-kit-bundle`) as a public repo, contract
  first, Packagist registered by the owner before the first tag. WorkflowBundle stays the state machine and is
  reached through an AutomationKit action that applies one of its transitions.
- **Consumers.** CrmKit block F-3 (0.24.0: CRM triggers and actions, sequences), Beacon spec 216 task automations
  (a candidate for migration, Q-AK-13), TaskBoard, Uptime and the target projects (spec 254 *Clarifications: Automation
  engine*: "TaskBoard, Uptime and the target projects need the same if X then Y").
- **Related owner decisions.** Q-254-6 (mail from automations and sequences only through MailKit `sending_domains`,
  SPF / DKIM / DMARC verified), D-20261123-01 (every dialog loads on demand: UiKit remote form dialog),
  D-20261117-15 (a final lock conflict answers 409 + `Retry-After`), 2026-11-10 (FrankenPHP worker mode with kernel
  reset false), 2026-11-11 (ARCH-001 rule 3 strict: own VO per port), 2026-11-18 (REQ-RUNTIME-003 concurrency is a
  formal REQ), 2026-11-19 (house access pattern: REQ-UI-002 security keys + `{Feature}AccessChecker`), 2026-11-23
  (reuse or extend existing nowo bundles before creating a new one: §0).
- **House bars from the first PR.** `BUNDLES_FIX_RULES.md`: REQ-ARCH-001 (standard Symfony default, DDD opt-in, rule 3
  strict, private entity manager), REQ-RUNTIME-002 / -003, identifier columns binary NO PAD (audit LQ), versioned
  public HTTP paths, runtime settings env / DB where it applies, UiKit + FormKit for every page and demo, 100 % of the
  applicable REQs of `BUNDLES_FULL_SPECS_DETAILS.md` before 1.0.0, and the 221 conventions (UUIDv7 `string(36)` binary
  ids, XML mapping default + attribute twin, public HTTP APIs disabled by default).

Requirements data read for this contract (2026-11-23):

| Source | Version read | What was read |
| --- | --- | --- |
| Beacon spec 254 (read-only) | worktree `agent-ab8ef200932dd357a`, `e668c522d` | `specs/254-crm-hubspot-parity/spec.md`: inventory, *Clarifications*, cross-block FR-001 – FR-007, **block F FR-060 – FR-064**, SC-005, out of scope |
| CrmKit roadmap | PR #24, branch `docs/roadmap-spec-254`, `4fc258f0c` | `docs/ROADMAP.md`, *HubSpot parity*: F-0, F-1, F-2, F-3, Block F *Depends on / New ports / New VOs / Immediate audit*, Q-254-1 … 14 |
| Beacon spec 216 (prior art, read-only) | symfony-beacon `1f5e64403` | `specs/216-simple-task-automations/spec.md`, `src/Tasks/**/Automation/*` (≈ 3,360 lines: rule, run, catalogue, cycle detector, chain context, run handler, due-passed trigger, purge) |
| WorkflowBundle | `main` `6c313ae` (v2.3.0) | `src/Entity/*`, `Contract/WorkflowRuntimeInterface`, `Expression/GuardExpression`, `Service/WorkflowRuntime`, `Tenancy/*`, `Event/WorkflowTransitionAppliedEvent`, docs `MULTITENANCY.md`, `USAGE.md`, `CONCURRENCY.md` |
| Sibling bundles (ports reused by actions) | `main` / latest release | MailKit 2.2 (`Sender/SenderIdentityProviderInterface`, `Readiness/OutboundMailReadinessInterface`, `Log/*Eraser*`; **no `sending_domains` yet**: F-0), SmsKit 2.0 (`Sender/SmsSenderInterface::send(SmsMessage): SmsSendResult`), NotifyKit 0.3 (`Contract/NotifierInterface::notify(NotificationRequest)`), TaskBoard 2.31 (`Api/Operation/TaskApiTaskOperations::createTask()`), CrmKit 0.8 (`Api/Operation/*`), OutboundUrlGuardBundle (`Guard/OutboundUrlGuard`), transaction-doctrine 1.9 (write units), runtime-setting-contract 1.0 |
| Earlier contracts (style) | — | NotifyKitBundle `docs/CONTRACT.md` (221-B), WorkingTimeBundle `docs/CONTRACT.md`, ApiToken (221-A), ReportKit (221-C) |

Everything in the sources above is treated as requirements data only.

---

## 0. Why not extend WorkflowBundle

Owner rule 2026-11-23: reuse or extend an existing nowo bundle before creating a new one. Q-254-7 already listed
"(c) extend WorkflowBundle" and chose "(a) new repo", but the rule asks for a concrete check. This section checks
WorkflowBundle **2.3.0** (`main` `6c313ae`, 148 PHP files) against every capability spec 254 block F needs.

**What WorkflowBundle is.** Database-stored definitions for the Symfony Workflow component: `WorkflowDefinition`
(slug, type state machine / workflow, subject class, marking property, match rules, tenant clones and versions),
`WorkflowPlace`, `WorkflowTransition` (from / to places, `guardPermission`, `guardExpression`, `automatic`), and
`WorkflowSubjectPin` (which definition version a subject runs). At run time `WorkflowRuntimeInterface::apply($subject,
$slug, $transition)` checks the guards, changes the **marking property of the host's own object** and follows
automatic transitions in the same call. It never persists the subject (the host does). It emits
`WorkflowTransitionAppliedEvent`.

| Capability block F needs | In WorkflowBundle 2.3 | What extending would take |
| --- | --- | --- |
| **Triggers** (record created / updated, stage entered, field changed, form submitted, SLA breached, *no activity for N days*) | None. A transition happens only when host code calls `apply()`. Nothing listens to domain events or scans for time conditions | An event intake, a trigger registry, per-tenant matching tables and a scheduled scanner: new subsystems with no link to places or markings |
| **Conditions** | Yes, in a narrow form: `guardPermission` and `guardExpression`, a closed DSL (`GuardExpression`, `@internal`) over whitelisted subject variables, evaluated synchronously against the object being moved | Reusable as an idea (§6.3, Q-AK-3). But guards gate a move of one object, while automation conditions read other data (segments, last activity, consent) through host evaluators |
| **Actions** (create task, set field, notify, send mail, webhook, enrol, apply a transition) | None. A transition's only effect is the new marking. Side effects are host listeners on Symfony Workflow events, synchronous, inside the caller's request | An action registry, per-action idempotency keys, permission re-checks as the author, retries, an outbox for external effects: the core of a different bundle |
| **Delays and wait-until** | None. No timers and no Messenger. Automatic transitions fire in the same call | Due-time rows, a scheduler tick, leases, business-day windows, time zones and DST |
| **Runs and run log** (idempotent per event, statuses, audit, dry run) | None. There is no execution record: audit is the marking change | Run and step tables, unique (rule, event) keys, retention, operator pages |
| **Enrolments and sequences** (one record in N timed steps, stop on reply / booking / unsubscribe, daily caps) | Partly modellable: steps as places, *next* as automatic transitions, stop as a transition. But the marking lives **on the host's object** (one property per workflow), so a contact in two sequences needs two marking columns on the host's contact. Re-enrolment, per-enrolment timers and caps do not exist | An enrolment table holding the marking outside the subject (a new marking store), plus timers, caps and stop signals: everything except the place names |
| **Loop and recursion guards, rate limits, kill switch** | Not needed by a synchronous state machine (automatic transitions stop when no transition is enabled) | Chain context across Messenger messages, cycle detection over trigger → action edges, throttles |
| **Lead scoring** | Not related | Not in AutomationKit either: scoring stays in CrmKit F-1 (§1); AutomationKit consumes a *score band changed* trigger |
| **Tenancy** | Yes: tenant clones of global templates, versions, subject pins (`MULTITENANCY.md`) | Reused as a pattern (§8) |

**Verdict: extending WorkflowBundle is not viable.** Only conditions (partly) and tenancy exist there. Triggers,
actions, delays, runs, enrolments and safety limits would all be new, and their data model (an event-driven, durable,
asynchronous run with side effects) has nothing in common with Workflow's (the synchronous marking of one host
object). Putting them into WorkflowBundle would:

1. turn a stable 2.x state-machine bundle (used by WorkingTime leave approval, Beacon requests, header search) into a
   second product, with its schema, audits and release cadence tied to CRM work;
2. force every Workflow host to carry Messenger, scheduler and outbox code paths, even with automation disabled;
3. break the Symfony Workflow contract that the marking lives on the subject, which sequences cannot respect.

So this contract builds a new bundle and **reuses** WorkflowBundle where it fits: the action
`workflow.apply_transition` calls `WorkflowRuntimeInterface::apply()` (Workflow keeps owning guards, versions and
pins), a Workflow bridge turns `WorkflowTransitionAppliedEvent` into the trigger `workflow.transition_applied`, and
the tenancy model mirrors Workflow's (global templates copied per tenant, runs pinned to a revision). Because the
check shows extending is not viable, there is no "build or extend" question in §21. The owner's Q-254-7 answer (a)
stands.

### 0.1 Existing bundles the actions reuse (nothing re-implemented)

| Action / feature | Reused bundle and port | Notes |
| --- | --- | --- |
| `mail.send` (templated mail to the subject) | **MailKit**: Symfony Mailer through MailKit's queue and log, sender identity, **`sending_domains`** (F-0, to be built in MailKit) | Refused unless the from-domain is verified for the tenant (Q-254-6, §13) |
| Sequence e-mail from the enrolling person's mailbox | **CrmKit** action over MailKit B1 / B2 mailboxes (`CrmMailboxSenderInterface`) | A CrmKit action under the same engine-level mail rules (§13) |
| `sms.send` | **SmsKit** `SmsSenderInterface::send(SmsMessage)` | 1.x bridge, off by default, consent mandatory (Q-AK-11) |
| `notify` (bell / push / Slack) | **NotifyKit** `NotifierInterface::notify(NotificationRequest)` | NotifyKit keeps preferences, quiet hours and channels |
| `taskboard.create_task`, `taskboard.add_comment` | **TaskBoard** `Api/Operation/TaskApiTaskOperations` as the run's actor | The TaskBoard bridge also provides task triggers (Beacon spec 216 catalogue) |
| `workflow.apply_transition` | **WorkflowBundle** `WorkflowRuntimeInterface::apply()` | `WorkflowConflictException` → retry; a blocked transition → `skipped` with reason |
| `crm.set_field`, `crm.add_tag`, `crm.assign_owner` (round-robin in a team), `crm.create_next_action`, sequence e-mail | **CrmKit** `Api/Operation/*` in CrmKit's `Bridge/AutomationKit` (CrmKit roadmap F-3) | Shipped by CrmKit, not by this bundle: AutomationKit has no CRM dependency |
| `http.webhook` | **OutboundUrlGuardBundle** (`OutboundUrlGuard`: pinned IP, no redirects, timeouts, byte caps) | Built in, off by default (Q-AK-14) |
| Business-day windows | **WorkingTimeBundle** holiday calendars (optional bridge) | Weekdays and hours without it (Q-AK-15) |
| Write units, 409 + `Retry-After` | **transaction-doctrine** / **transaction-contract** | §11 |
| Runtime settings, kill switch | **runtime-setting-contract** (+ InstanceSettings bridge) | §12.3 |
| Erasure / export | **UserKit** `PersonalData*` | §14 |
| Audit | **AuditKit** | §12.4 |
| Public API auth | **ApiTokenBundle** module `automations` | §16 |
| Pages and forms | **UiKit**, **FormKit** | §15 |

## 1. Scope and non-goals

**In (generic, no CRM dependency):**

- **Automations (rules).** "When a trigger fires, if conditions hold, run these steps": a trigger with parameters,
  a condition tree, then an ordered list of steps (actions, delays, wait-until, condition gates, stop).
- **Sequences.** Automations of kind `sequence`: a subject is **enrolled** (by an action, by hand, in bulk or through
  the API) and walks timed steps with execution windows, daily caps, stop rules and a re-enrolment policy.
- **Registries by service tag.** Trigger sources, condition evaluators, action handlers, subject resolvers, choice
  providers for parameter forms; a compiled, immutable catalogue.
- **Execution on Messenger**, idempotent per event and per step, with retries, durable delays, wait-until, leases,
  REQ-RUNTIME-003 concurrency rules and FrankenPHP worker safety.
- **Safety:** cycle check on save, chain depth and visited set at run time, rate limits and throttles, per-subject
  caps, per-action and global kill switches, a circuit breaker on repeated failures, author re-check at run time.
- **Run log and audit:** runs and steps with statuses and reasons (ids only, no content), retention, AuditKit bridge.
- **Dry run** on sample subjects: no writes, every action's `simulate()`.
- **UI** on UiKit + FormKit: list, editor with step dialogs loaded on demand, run log, enrolments, operator page.
- **GDPR:** subject and actor erasure and export, retention.
- **Application operations** (`Api\Operation`) for the optional public API `<prefix>/automations/v1`, and for host MCP
  tools.
- **Bridges** for the actions and triggers of §0.1.

**Non-goals (v1):**

- **Lead scoring.** It stays in CrmKit F-1 (explainable point rules on CRM fields, bands, breakdown). Scores are CRM
  data, not runs. AutomationKit reacts to CrmKit's `crm.score_band_changed` trigger.
- **The state machine.** WorkflowBundle (§0).
- **CRM triggers and actions**, segments, consent storage, unsubscribe pages, mailbox linking: CrmKit / MailKit.
- Visual flow-chart editor with free-form graphs and parallel branches (an ordered step list with gates in 1.0,
  Q-AK-2), A/B splits, AI-written steps (spec 254 out of scope), goal-based exits beyond stop rules, cross-tenant
  automations, user-written code or expressions with function calls, inbound webhooks as triggers (hosts map their
  own inbound events to triggers), SMS in 1.0 (Q-AK-11).

## 2. Package

| Item | Proposal |
| --- | --- |
| Name | `AutomationKitBundle`, Composer `nowo-tech/automation-kit-bundle`, GitHub `nowo-tech/AutomationKitBundle` (public, MIT). The owner registers it on Packagist once S0 is on `main`, before the first tag |
| Namespace / class | `Nowo\AutomationKitBundle` / `NowoAutomationKitBundle`; config root `nowo_automation_kit`; Twig namespace `@NowoAutomationKit`; translation domain `NowoAutomationKitBundle`; Stimulus prefix `nowo-automation-kit--`; route names `nowo_automation_kit_*` |
| PHP / Symfony | `>=8.2 <8.6`; `^7.4 \|\| ^8.0`; demos Symfony 7.4 + 8.1 on FrankenPHP worker, MySQL, Messenger Doctrine transport |
| Runtime deps | `doctrine/dbal ^3.9 \|\| ^4`, `doctrine/orm ^3.3` (private EM, mapping), `doctrine/doctrine-bundle`, `symfony/{framework-bundle,http-kernel,routing,config,dependency-injection,event-dispatcher,messenger,scheduler,clock,uid,rate-limiter,translation,form,validator,twig-bundle,console}`, `psr/log`, `psr/clock`, `nowo-tech/transaction-contract ^1.0`, `nowo-tech/transaction-doctrine ^1.9`, `nowo-tech/runtime-setting-contract ^1.0`, `nowo-tech/ui-kit-bundle`, `nowo-tech/form-kit-bundle` |
| Suggest | `nowo-tech/mail-kit-bundle` (`mail.send`, needs `sending_domains`), `nowo-tech/notify-kit-bundle`, `nowo-tech/task-board-bundle`, `nowo-tech/workflow-bundle ^2.3`, `nowo-tech/sms-kit-bundle` (1.x), `nowo-tech/outbound-url-guard-bundle` (`http.webhook`), `nowo-tech/working-time-bundle` (holidays), `nowo-tech/user-kit-bundle`, `nowo-tech/audit-kit-bundle`, `nowo-tech/api-token-bundle`, `nowo-tech/instance-settings-bundle` |
| Why Messenger and Scheduler are required | Every run is asynchronous by design (§11). A synchronous fallback would run actions inside the producer's request, against FR-061. Hosts without a worker can use the Doctrine transport and `messenger:consume` |
| First tag | **0.1.0** after S1 (Q-AK-17); 0.x per slice; **1.0.0** after the REQ audit (`bundle-req-audit` 100 %) and the SEC-004 audits |

## 3. Shape

```
producer (host code / bridge listener)                           host UI / API / MCP
   │ AutomationEventsInterface::record(TriggerEvent)                │ enrol · pause · dry run · edit
   ▼                                                                ▼
after-commit buffer (dropped on rollback) ──▶ MatchEventMessage ──▶ Matcher (tenant, trigger key, params)
                                                                    │ INSERT run UNIQUE(automation, event_key)
                                                                    ▼
Scheduler tick (enqueue only) ──▶ DispatchDueStepsMessage ──▶ claim due steps (conditional UPDATE, lease)
                                                                    │
                                                                    ▼
                                          ExecuteStepMessage ──▶ StepExecutor
   kill switches ─ throttle ─ chain guard ─ author re-check ─ subject re-check (tenant, exists) ─ conditions
                                                                    │
            ┌───────────────┬───────────────┬───────────────┬───────┴────────┬──────────────────┐
         action          delay          wait_until        gate (if)        stop / enrol / unenrol
   ActionHandler (tag)   due_at=…      waiting + signal   continue / end
   simulate · execute
      │ mail.send (MailKit + sending_domains) · notify (NotifyKit) · taskboard.* · workflow.apply_transition
      │ http.webhook (OutboundUrlGuard) · crm.* (CrmKit bridge) · host actions
      ▼
   step outcome (write unit) ─▶ next step due ─▶ run done / failed / stopped ─▶ events after commit (ids only)
```

The bundle owns automations, revisions, runs, steps, enrolments, the catalogue, the execution machinery, the safety
limits, the run log, the pages and the jobs. The host (or a bridge) owns what a subject is, whether it exists and
belongs to the tenant, who may do what, the trigger events it records, its own actions and conditions, and where the
UI is mounted.

## 4. Layout and architecture rules

```
src/Domain/{Automation,Run,Sequence,Catalog,Shared}/   VOs, enums, entities (plain PHP), ports, events, exceptions
src/Application/{Authoring,Intake,Execution,Sequence,DryRun,Retention,Api}/   services over the ports
src/Infrastructure/Doctrine/      private EM `nowo_automation_kit`, XML mapping (default) + attribute twins, DBAL stores
src/Infrastructure/Symfony/       DI, config tree, Messenger handlers, Scheduler recipe, controllers, FormKit forms, Twig
src/Bridge/{MailKit,NotifyKit,TaskBoard,Workflow,OutboundUrlGuard,WorkingTime,UserKit,AuditKit,ApiToken,InstanceSettings,SmsKit}/
src/Resources/{config,views,translations,public}/
```

- **Rule 3, strict (ARCH-001):** public ports, VOs and events use bundle VOs and scalars only: no Doctrine, Symfony,
  HTTP or host type. Each port has its own VOs (no shared "bag" type across ports, owner 2026-11-11).
- **Architecture test** (`tests/Architecture/LayerDependencyTest`): `Domain` imports no framework class; bridges are
  the only code importing optional packages; `Application` never imports `Infrastructure`.
- **Both host kinds (REQ-ARCH-001):** standard Symfony hosts get default resolvers over Doctrine entities; DDD hosts
  plug adapters over their read models. The same functional suite runs on both fixtures (§20).
- **Private entity manager** `nowo_automation_kit`; the bundle never flushes, clears or resets a host EM (rule 6).
- No host entity relation, no `resolve_target_entities`, no FK to host tables: references are `type + id` strings.

## 5. Domain model

### 5.1 Concepts

| Concept | What it is |
| --- | --- |
| **Automation** | A tenant-owned definition: kind (`rule` \| `sequence`), name, optional scope (`ContextRef`: a board, a pipeline, a monitor group), trigger (rules only), entry conditions, ordered steps, settings (limits, failure policy, re-entry), status (`draft`, `active`, `paused`, `archived`), pause reason, **author** (the actor it runs as), optimistic `version`, current **revision** |
| **Revision** | An immutable snapshot of trigger + conditions + steps, created on every activation of changed content. Runs and enrolments pin the revision they started on; edits never change a running run's steps (Workflow's *pin* idea, §0) |
| **Trigger** | A registered event kind (`TriggerDefinition`: key, subject type, parameter schema, `canBeCausedBy` edges) and its occurrence (`TriggerEvent`: event key, tenant, subject, occurred at, scalar attributes, chain) |
| **Condition** | A tree of `all` / `any` / `not` nodes whose leaves call a registered `ConditionEvaluator` (key + params). Evaluated at entry and in gate and wait-until steps, always at run time, against fresh data |
| **Action** | A registered effect (`ActionDefinition`: key, parameter schema, required permission, effect class `internal` \| `external` \| `mail` \| `message`, `canCause` trigger edges, simulate support) run by an `ActionHandler` |
| **Step** | One entry of an automation's step list: `action`, `delay`, `wait_until`, `gate` (if not → end or skip to a label), `enrol`, `unenrol`, `stop`. Up to `limits.max_steps` (50) |
| **Run** | One execution of one automation revision for one subject: from a trigger event (rules) or an enrolment (sequences). Unique per (automation, event key). Status: `pending`, `running`, `waiting`, `done`, `skipped`, `denied`, `failed`, `stopped`, `throttled`, `cancelled` |
| **Run step** | The record of one step of a run: index, kind, status, attempts, idempotency key, due at, lease, outcome code. The unit of claiming, retrying and logging |
| **Enrolment** | A subject in a sequence: enrolled by (actor or run), revision, status (`active`, `paused`, `completed`, `stopped`), stop reason, current step, next due at. At most one **active** enrolment per (sequence, subject) |
| **Chain** | The causal path of runs: chain id, depth, visited automation ids. A run's actions record events that inherit the chain one level deeper (Beacon spec 216 FR-008) |

### 5.2 Value objects and enums (public, `Nowo\AutomationKitBundle\Contract\*`)

```php
final readonly class TenantRef   { public function __construct(public string $id) {} public static function default(): self; } // 1–128 bytes, exact, no control/format chars
final readonly class SubjectRef  { public function __construct(public string $type, public string $id) {} public function key(): string; public function equals(self $o): bool; } // type ^[a-z][a-z0-9_.]{0,63}$, id 1–128 bytes
final readonly class ActorRef    { public function __construct(public string $type, public string $id) {} }   // same rules; type 'user' by default
final readonly class ContextRef  { public function __construct(public string $type, public string $id) {} }   // the scope of an automation (board, pipeline…)
final readonly class AutomationId { public function __construct(public string $value) {} }                // UUIDv7
final readonly class RunId        { public function __construct(public string $value) {} }
final readonly class EnrolmentId  { public function __construct(public string $value) {} }

enum AutomationKind: string   { case Rule = 'rule'; case Sequence = 'sequence'; }
enum AutomationStatus: string { case Draft = 'draft'; case Active = 'active'; case Paused = 'paused'; case Archived = 'archived'; }
enum PauseReason: string      { case Manual = 'manual'; case AuthorNoAccess = 'author_no_access'; case InvalidParameter = 'invalid_parameter';
                                case CircuitBreaker = 'circuit_breaker'; case SendingDomain = 'sending_domain'; case CatalogMissing = 'catalog_missing'; }
enum RunStatus: string        { /* pending running waiting done skipped denied failed stopped throttled cancelled */ }
enum StepKind: string         { case Action = 'action'; case Delay = 'delay'; case WaitUntil = 'wait_until'; case Gate = 'gate';
                                case Enrol = 'enrol'; case Unenrol = 'unenrol'; case Stop = 'stop'; }
enum EffectClass: string      { case Internal = 'internal'; case External = 'external'; case Mail = 'mail'; case Message = 'message'; }

/** Scalars only; ≤ 30 keys; key ^[a-z][a-z0-9_]{0,63}$; string values ≤ 1,000 chars (text params ≤ 10,000). One class per use, not shared across ports. */
final readonly class TriggerAttributes { /** @param array<string, string|int|float|bool|null> $values */ public function __construct(public array $values) {} }
final readonly class ActionParams      { /** @param array<string, string|int|float|bool|null|list<string>> $values */ public function __construct(public array $values) {} }
final readonly class ConditionParams   { /* same shape as ActionParams */ }

final readonly class ChainRef { public function __construct(public string $chainId, public int $depth, public array $visited) {} } // visited: list<string> automation ids

final readonly class TriggerEvent
{
    public function __construct(
        public TenantRef $tenant, public string $triggerKey, public SubjectRef $subject, public string $eventKey,  // eventKey: stable producer id (≤ 128), the idempotency key of intake
        public \DateTimeImmutable $occurredAt, public TriggerAttributes $attributes, public ?ActorRef $actor = null,
        public ?ContextRef $context = null, public ?ChainRef $chain = null,  // chain: set by the engine, never by producers
    ) {}
}

final readonly class Duration       { public static function fromIso(string $iso8601): self; public function seconds(): int; } // PT1M … P365D
final readonly class ExecutionWindow                                                                      // sequences and delays
{
    /** @param list<int> $weekdays ISO 1–7 */
    public function __construct(public string $timezone, public array $weekdays, public string $from, public string $until, public bool $skipHolidays = false) {}
}
final readonly class ConditionNode  { /* op: all|any|not|leaf; children: list<self>; leaf: evaluator key + ConditionParams; max depth 8, ≤ 50 leaves */ }
final readonly class StepDefinition { public function __construct(public StepKind $kind, public ?string $label, public ?string $actionKey, public ?ActionParams $params,
                                      public ?Duration $delay, public ?ExecutionWindow $window, public ?ConditionNode $condition, public ?Duration $timeout,
                                      public ?string $onFalse /* 'end' | 'continue' | label */, public ?string $sequenceId) {} }
final readonly class StopRule       { public function __construct(public string $triggerKey, public ?ConditionNode $condition = null) {} } // e.g. crm.contact_replied, crm.meeting_booked
final readonly class SequenceSettings
{
    public function __construct(public ExecutionWindow $window, public ?int $dailyCapPerSender, public array $stopRules,          // list<StopRule>
                                public ReEnrolment $reEnrolment /* never | after_completion | always */, public ?Duration $reEnrolCooldown) {}
}

final readonly class ActionContext   // what a handler receives
{
    public function __construct(public TenantRef $tenant, public SubjectRef $subject, public ActorRef $runAs, public ActionParams $params,
                                public string $idempotencyKey,   // sha256(run id | step index); identical on every retry of this step
                                public RunId $runId, public ChainRef $chain, public TriggerAttributes $trigger, public bool $dryRun) {}
}
enum OutcomeKind: string { case Done = 'done'; case NoChange = 'no_change'; case Skipped = 'skipped'; case FailedPermanent = 'failed'; case Retry = 'retry'; }
final readonly class ActionOutcome { public function __construct(public OutcomeKind $kind, public ?string $code = null, public ?string $resultRef = null, public ?int $retryAfterSeconds = null) {} } // code ≤ 64 ascii, resultRef: an id the action created (task id), never content
final readonly class Simulation    { public function __construct(public bool $wouldRun, public string $summaryKey, public array $summaryParams, public ?string $refusalCode = null) {} }
final readonly class ConditionResult { public function __construct(public ?bool $value, public ?string $reasonCode = null) {} } // null = unknown → treated as false, logged
```

### 5.3 Tables (bundle-owned, prefix `doctrine.table_prefix`, default `nowo_`; `<p>` below)

Conventions: ids `string(36)` UUIDv7, binary; host refs (`tenant_id`, `subject_type`, `subject_id`, `actor_*`,
`context_*`) binary NO PAD (`utf8mb4_bin` / `ascii_bin` on MySQL, exact on PostgreSQL and SQLite, asserted by a
test), and every read by a host ref re-checks with `===` in PHP (audit LQ). Times `datetime_immutable` UTC.

| Table | Columns (main) | Keys and indexes |
| --- | --- | --- |
| `<p>automation` | `id`, `tenant_id`, `kind`, `name` (120), `description` (500) NULL, `context_type/id` NULL, `status`, `pause_reason` NULL, `author_type/id` NULL, `current_revision` (int), `draft` (json: trigger, conditions, steps, settings), `failure_streak`, `last_run_at`, `last_run_status`, `template_key` NULL, **`version`**, `created_at`, `updated_at` | idx `(tenant_id, status, kind)`, `(tenant_id, context_type, context_id)` |
| `<p>automation_revision` | `automation_id`, `revision`, `trigger_key` NULL, `trigger_params` (json), `conditions` (json), `steps` (json), `settings` (json), `catalog_hash`, `activated_by_*`, `created_at` | PK `(automation_id, revision)`; **INSERT only** |
| `<p>automation_trigger_index` | `tenant_id`, `trigger_key`, `automation_id`, `revision`, `match_params` (json) | PK `(tenant_id, trigger_key, automation_id)`: one-query matching of active rules; rewritten on activate / pause in the same write unit |
| `<p>automation_run` | `id`, `tenant_id`, `automation_id`, `revision`, `kind`, `event_key` (128 bin), `trigger_key`, `subject_type/id`, `enrolment_id` NULL, `chain_id`, `depth`, `status`, `reason_code` NULL, `current_step`, `run_as_*` NULL, `created_at`, `finished_at` NULL | **unique `(automation_id, event_key)`** (idempotent intake); idx `(tenant_id, created_at)`, `(tenant_id, subject_type, subject_id)`, `(automation_id, status)` |
| `<p>automation_run_step` | `id`, `run_id`, `tenant_id`, `step_index`, `kind`, `status` (`pending`, `due`, `claimed`, `dispatching`, `waiting`, `done`, `skipped`, `failed`, `cancelled`), `due_at` NULL, `wait_until_at` NULL, `lease_owner` NULL, `lease_until` NULL, `attempts`, `idempotency_key` (64 ascii), `outcome_code` NULL, `result_ref` NULL, `updated_at` | **unique `(run_id, step_index)`**, **unique `idempotency_key`**; idx `(status, due_at)` (tick), `(tenant_id, status)`, `(status, wait_until_at)` |
| `<p>automation_wait` | `run_step_id`, `tenant_id`, `subject_type/id`, `signal_keys` (json) | idx `(tenant_id, subject_type, subject_id)`: events wake waiting steps of the same subject |
| `<p>automation_enrolment` | `id`, `tenant_id`, `sequence_id`, `revision`, `subject_type/id`, `status`, `stop_reason` NULL, `enrolled_by_*` NULL, `enrolled_by_run` NULL, `sender_ref` NULL (mailbox / user the steps send as), `run_id`, `active_key` (NULL unless active: `sequence_id|subject key`), `enrolled_at`, `ended_at` NULL, **`version`** | **unique `active_key`** (one active enrolment per sequence × subject; NULLs ignored); idx `(tenant_id, subject_type, subject_id)`, `(sequence_id, status)` |
| `<p>automation_counter` | `tenant_id`, `bucket` (e.g. `runs:tenant:2026-11-23T10`, `mail:sender:<ref>:2026-11-23`), `count`, `expires_at` | PK `(tenant_id, bucket)`; `UPDATE … SET count = count + 1 WHERE count < :limit` (atomic caps; §12.2) |
| `<p>automation_stats_daily` | `tenant_id`, `automation_id`, `day`, counts per status | PK `(tenant_id, automation_id, day)`; survives run retention and subject erasure (no subject) |

Run steps and waits never store rendered content, mail bodies, webhook payloads or subject field values: only codes
and ids. Action params stay in the revision (tenant configuration, §14).

## 6. Public ports (bundle-implemented services)

```php
interface AutomationEventsInterface
{
    /** Records one trigger occurrence. Inside a transaction on the bundle connection or the host's: held until the outermost commit, dropped on rollback. Never throws for an unknown trigger key (logged, counted); validation errors throw InvalidTriggerEvent. */
    public function record(TriggerEvent $event): void;
    /** Signals for wait-until steps and stop rules without starting rules (e.g. "reply received"). Same buffering. */
    public function signal(TriggerEvent $event): void;
}

interface SequencesInterface
{
    public function enrol(TenantRef $t, AutomationId $sequence, SubjectRef $s, ActorRef $by, ?string $senderRef = null): EnrolResult; // enrolled | already_active | refused(reason)
    /** @param list<SubjectRef> $subjects ≤ 1,000; chunked async */
    public function enrolMany(TenantRef $t, AutomationId $sequence, array $subjects, ActorRef $by, ?string $senderRef = null): BulkEnrolResult;
    public function unenrol(TenantRef $t, EnrolmentId $id, ActorRef $by, string $reason = 'manual'): bool;
    public function stopFor(TenantRef $t, SubjectRef $s, string $reason): int;          // e.g. unsubscribe: stops every active enrolment of the subject
    public function enrolmentsOf(TenantRef $t, SubjectRef $s): array;                   // list<EnrolmentView>
}

interface AutomationAdminInterface  // what the UI, the API and MCP operations use; each call checks access (§9)
{
    public function create(TenantRef $t, ActorRef $by, AutomationDraft $draft): AutomationId;
    public function update(TenantRef $t, AutomationId $id, int $expectedVersion, ActorRef $by, AutomationDraft $draft): void; // stale → 409
    public function activate(TenantRef $t, AutomationId $id, int $expectedVersion, ActorRef $by): ActivationResult;           // validation + cycle check + catalogue + mail domain check
    public function pause(TenantRef $t, AutomationId $id, ActorRef $by): void;
    public function archive(TenantRef $t, AutomationId $id, ActorRef $by): void;      // cancels pending steps, stops active enrolments (reason 'archived')
    public function dryRun(TenantRef $t, AutomationId $id, array $samples, ActorRef $by): DryRunReport; // ≤ 10 subjects, no writes, no external calls
}

interface RunLogInterface
{
    public function page(TenantRef $t, RunFilter $filter, ?string $after, int $limit): RunPage;   // keyset (REQ-PERF-001)
    public function run(TenantRef $t, RunId $id): ?RunView;                                       // with steps; null when not visible
    public function cancel(TenantRef $t, RunId $id, ActorRef $by): bool;
}

interface AutomationSubjectsInterface   // GDPR and lifecycle (§14)
{
    public function forgetSubject(TenantRef $t, SubjectRef $s): ForgetResult;
    public function forgetTenant(TenantRef $t): ForgetResult;
    public function forgetContext(TenantRef $t, ContextRef $c): int;                 // a board / pipeline deleted: archives its automations
}
```

## 7. Host extension points (SPIs; tagged, each with a safe default)

### 7.1 Trigger sources

| Interface | Signature | Notes |
| --- | --- | --- |
| `TriggerSourceInterface` (tag `nowo_automation_kit.trigger_source`) | `triggers(): iterable<TriggerDefinition>` | Declares keys, subject type, label keys, parameter schema (§7.5), `matches(TriggerParams $configured, TriggerAttributes $event): bool` hint (default: equality of declared match params), and `canBeCausedBy` (action key + params → bool) for the cycle check |
| `ScheduledTriggerSourceInterface` (same tag) | `due(TenantRef, string $triggerKey, TriggerParams $params, \DateTimeImmutable $now, ?string $cursor, int $limit): ScheduledBatch` | Time-based triggers ("no activity for N days", "due date passed", "SLA breached"): the bundle asks per active automation on the tick, the source returns subjects + stable event keys (e.g. `no_activity:<subject>:<last activity date>`), so a repeated tick never starts a second run |

Producers call `AutomationEventsInterface::record()` after their own write (in a listener, or inside their
transaction: the after-commit buffer covers it). Bridges shipped: Workflow (`workflow.transition_applied`), TaskBoard
(task created, moved to column, completed, reopened, member added, priority set, tag added, comment added, due passed:
the spec 216 catalogue), AutomationKit itself (`automation.enrolment_completed`, `automation.enrolment_stopped`). CrmKit
ships its own (record created / updated, stage entered, field changed, form submitted, ticket opened, SLA breached,
score band changed, no activity for N days, contact replied, meeting booked, unsubscribed).

### 7.2 Condition evaluators

| Interface | Signature | Default |
| --- | --- | --- |
| `ConditionEvaluatorInterface` (tag) | `definition(): ConditionDefinition` · `evaluate(TenantRef, SubjectRef, ConditionParams, EvaluationContext): ConditionResult` | Built in: `trigger.attribute` (compare an attribute of the triggering event), `subject.variable` (compare a whitelisted variable from `SubjectVariablesProviderInterface`), `time.window` (now within a window), `enrolment.active` (subject active in a sequence), `run.count` (times this automation ran for the subject) |
| `SubjectVariablesProviderInterface` (tag, per subject type) | `variables(): list<VariableDefinition>` · `values(TenantRef, SubjectRef, list<string> $names): array<string, scalar\|null>` | None: `subject.variable` lists nothing until a host or bridge declares variables (the Workflow `WorkflowSubjectVariablesProviderInterface` idea; a Workflow bridge adapter is possible) |

Evaluators are read-only and must not write (checked in dry-run tests with a write spy). Errors and unknown
answers count as `false` with a reason code in the step log.

### 7.3 Action handlers

| Interface | Signature | Contract |
| --- | --- | --- |
| `ActionHandlerInterface` (tag `nowo_automation_kit.action`) | `definition(): ActionDefinition` · `validate(TenantRef, ActionParams): list<ParamViolation>` · `simulate(ActionContext): Simulation` · `execute(ActionContext): ActionOutcome` | **Idempotent per `idempotencyKey`** (the same key on every retry). `internal` actions run inside the step's write unit on the bundle connection or their own; they return `Retry` for transient failures, never swallow them. `external` and `mail` actions are called after the step is marked `dispatching` (§11.4). Handlers never read the HTTP request or the security token: they act **as** `ActionContext::runAs`. Handlers with `EffectClass::Mail` are only accepted when the MailKit sending-domain guard is wired (§13) |

### 7.4 Subjects, actors, tenants, access

| Interface | Signature | Default |
| --- | --- | --- |
| `SubjectResolverInterface` (tag, per subject type) | `exists(TenantRef, SubjectRef): bool` · `label(TenantRef, SubjectRef, string $locale): ?string` · `url(TenantRef, SubjectRef): ?string` (local path) · `search(TenantRef, string $q, int $limit): list<SubjectChoice>` (dry-run and enrol pickers) · `canBeActedOnBy(TenantRef, SubjectRef, ActorRef, string $permission): bool` | None: an automation on a subject type without a resolver cannot be activated (`catalog_missing`) |
| `AutomationTenantResolverInterface` | `current(): TenantRef` (UI, API) | `SingleTenantResolver` (`TenantRef::default()`); multi-tenant hosts must configure one (container refuses `tenancy.mode: multi` without it) |
| `AutomationAccessCheckerInterface` | `can(ActorRef, TenantRef, AutomationPermission, ?ContextRef): bool`, permissions `view`, `manage`, `enrol`, `operate` (kill switch, operator page) | `RoleBasedAutomationAccessChecker` over the REQ-UI-002 security keys (`security.view_roles` …); deny by default; `allow_all` only for the demo |
| `ActorDirectoryInterface` | `isActive(ActorRef)`, `displayName(ActorRef)`, `actorOf(object $user): ActorRef` | UserKit adapter when installed, else the security user's identifier, active unless disabled |
| `ParamChoiceProviderInterface` (tag) | `choices(TenantRef, string $source, ?ContextRef, string $q): list<Choice>` | For parameter fields of kind `choice` with a dynamic source (columns, pipelines, templates, teams) |

### 7.5 Parameter schema (how forms are built without host form types)

`ParamSchema` is a list of `ParamField` VOs: `key`, `kind` (`string`, `text`, `int`, `bool`, `choice`, `multi_choice`,
`duration`, `window`, `subject_field`, `actor`, `template`, `url`), `required`, `labelKey`, `helpKey`, bounds,
`choiceSource` (static list or a `ParamChoiceProviderInterface` source). The bundle maps each kind to a FormKit field
type and validates server side with the same schema; handlers add their own `validate()`. Text params are never
evaluated as code; templates are referenced by key and rendered by their owner (MailKit, CrmKit).

## 8. Tenant model

- Every automation, revision, run, step, wait, enrolment and counter has a `tenant_id` (`TenantRef`). Single-tenant
  hosts use `default`.
- **Intake:** an event matches only automations of its own tenant (index `(tenant_id, trigger_key)`). A
  `TriggerEvent` with a subject of another tenant is a producer bug: the subject re-check (`exists(tenant, subject)`)
  refuses it at run time (`failed`, `subject_gone`), so a forged or wrong event cannot act across tenants.
- **Messages** carry ids only and are **signed** (`#[AsMessageHandler(sign: true)]` on Symfony 7.4+, else an HMAC
  stamp over `kernel.secret`), as Beacon spec 216 does: a forged queue message cannot make a rule act on another
  tenant's subject.
- **Access:** every UI, API and operation call resolves the tenant from `AutomationTenantResolverInterface` and looks
  rows up by id **and** tenant; a row of another tenant answers 404 (`REQ-REV-009`).
- **Scopes:** an automation may be scoped to a context inside the tenant (board, pipeline). Access is checked per
  context (`can(actor, tenant, manage, context)`); a scoped automation only matches events whose `context` equals its
  scope.
- **Templates:** bridges may ship **recipes** (read-only templates, e.g. "deal stalled 5 days → task + sequence").
  Creating from a recipe copies it into the tenant (`template_key` kept for upgrades), the Workflow pattern of global
  templates cloned per tenant. No row is shared between tenants.
- **Limits** are per tenant (§12.2), with per-tenant overrides through the settings provider.

## 9. Permissions (deny by default)

| Permission | Allows | Beacon mapping (proposal) |
| --- | --- | --- |
| `view` | Automations list, definitions, run log of the tenant / context | `crm.automation.manage` for CRM scopes; board managers for task scopes |
| `manage` | Create, edit, activate, pause, archive, dry run | `crm.automation.manage` (spec 254) |
| `enrol` | Enrol / unenrol subjects in sequences by hand or in bulk | `crm.marketing.manage` (sequences, spec 254) |
| `operate` | Operator page, kill switches, cancel runs of the tenant | Organization admins |

At **run time** the automation acts as its **author** (whoever activated it last, Beacon spec 216 FR-005 / FR-017).
Before every action step the engine checks: the author exists and is active (`ActorDirectoryInterface`), may still
`manage` the automation, and `canBeActedOnBy(subject, author, action.permission)`. Otherwise the step is `denied`, the
automation pauses (`author_no_access`) and its managers get one notice (NotifyKit when installed, else a log entry).
Q-AK-4 asks whether a system actor should be possible.

## 10. Configuration (strict tree, REQ-SF-006; defaults shown)

```yaml
nowo_automation_kit:
    enabled: false                       # nothing registered when false
    doctrine: { entity_manager: nowo_automation_kit, connection: default, table_prefix: 'nowo_', mapping: xml }
    tenancy: { mode: single, resolver: null }
    access: { checker: role_based, view_roles: ['ROLE_ADMIN'], manage_roles: ['ROLE_ADMIN'], enrol_roles: ['ROLE_ADMIN'], operate_roles: ['ROLE_ADMIN'] }
    messenger:
        transport_hint: async            # docs only: routing stays in the host's messenger.yaml
        max_seconds: 50                  # handlers re-dispatch the remainder (REQ-MESSENGER-009)
        step_batch: 200                  # due steps claimed per tick message
        lease_seconds: 300
    schedule: { enabled: false, tick: '* * * * *', scheduled_triggers: '*/15 * * * *', retention: '20 4 * * *' }
    execution:
        max_attempts: 5                  # per step; backoff 1, 5, 25, 120 min (+ jitter)
        failure_policy: stop_run         # stop_run | continue (per automation override)
        circuit_breaker: { consecutive_failures: 10 }   # pauses the automation (circuit_breaker)
        delay_max: P365D
        wait_timeout_max: P90D
    limits:
        max_chain_depth: 3               # 1–10
        max_active_automations_per_tenant: 100
        max_active_automations_per_context: 25
        max_steps: 50
        runs_per_tenant_hour: 500        # above: run 'throttled' (Beacon spec 216 FR-011)
        runs_per_automation_subject_day: 10
        bulk_enrol_max: 1000
        dry_run_samples_max: 10
    kill_switch:
        global: false                    # runtime setting (env / DB); true = intake records nothing, ticks claim nothing
        actions_disabled: []             # e.g. ['mail.send', 'http.webhook']
    sequences:
        default_window: { timezone: UTC, weekdays: [1, 2, 3, 4, 5], from: '09:00', until: '17:00', skip_holidays: false }
        daily_cap_per_sender: 200
        re_enrolment: never
    mail: { require_sending_domain: true }   # cannot be false while mail actions exist (§13)
    retention: { runs_days: 30, ended_enrolments_days: 90, stats_days: 400, batch_size: 500 }
    bridges: { mail_kit: auto, notify_kit: auto, task_board: auto, workflow: auto, outbound_url_guard: auto, working_time: auto, user_kit: auto, audit_kit: auto, api_token: auto, instance_settings: auto, sms_kit: false }
    actions: { http_webhook: { enabled: false, timeout_seconds: 10, max_response_bytes: 65536 } }
    web_ui: { enabled: false, path_prefix: '/automations', layout_template: null, templates: {} }
    admin_ui: { enabled: false, path_prefix: '/admin/automations' }
    api: { enabled: false, prefix: '/api' }
    settings_provider: null              # runtime-setting-contract provider id
```

Every value that an operator may change at run time (kill switches, limits, retention) is a runtime setting
(env → DB switchable per setting, disableable by command, read per call, never cached in a service property).

## 11. Execution

### 11.1 Messages (ids only, signed)

| Message | Handler work |
| --- | --- |
| `MatchEventMessage` (tenant, trigger key, event key, subject, attributes, chain) | Loads matching active automations from the trigger index (one query); per automation: scope check, `runs_per_automation_subject_day` cap, `INSERT` the run with **unique `(automation_id, event_key)`** (a duplicate is ignored: idempotent intake), insert step 0 as `due` |
| `DispatchDueStepsMessage` (tick) | Claims ≤ `step_batch` due steps (§11.3) and dispatches one `ExecuteStepMessage` each |
| `ExecuteStepMessage` (run step id, lease token) | Runs the step (§11.2) |
| `ScanScheduledTriggersMessage` (automation id, cursor) | Calls the `ScheduledTriggerSourceInterface` and records events with their stable keys |
| `WakeWaitingStepsMessage` (tenant, subject, signal key) | Re-evaluates waiting steps of the subject; stop rules of active enrolments |
| `BulkEnrolMessage` (≤ 200 subjects) | Enrols in chunks |
| `PruneMessage` (cursor) | Retention (§14.3) |

The Scheduler recipe only **enqueues** (`REQ-MESSENGER-008`); commands `nowo:automation-kit:tick`,
`nowo:automation-kit:scan`, `nowo:automation-kit:prune` serve hosts with their own maintenance schedule.

### 11.2 One step, in this order (each ends with one step row; Beacon spec 216 FR-005 / FR-008 – FR-011 generalised)

1. Lease still ours, step still `claimed`, run not `cancelled`; else nothing.
2. **Kill switches:** global on → release the claim, step back to `due` (held, Q-AK-8); action key disabled → same.
3. Automation `active` (a `paused` or `archived` automation: step `cancelled`, run `cancelled`).
4. **Subject re-check:** `exists(tenant, subject)`; gone → `failed` (`subject_gone`), enrolment stopped.
5. **Chain guard:** depth > `max_chain_depth` or automation already in `visited` → run `stopped` (`depth` / `loop`).
6. **Throttle:** tenant runs this hour ≥ `runs_per_tenant_hour` (atomic counter) → run `throttled` (start steps only).
7. **Author re-check** (§9) → `denied` + pause.
8. Step kind: `gate` / `wait_until` evaluate conditions; `delay` computes `due_at`; `action` runs §11.4; `enrol` /
   `unenrol` / `stop` call the sequence service.
9. Outcome, next step `due` (or run end), counters, stats and after-commit events in **one write unit**.

### 11.3 Claiming and leases (REQ-RUNTIME-003)

- Claim: `UPDATE <p>automation_run_step SET status='claimed', lease_owner=:token, lease_until=:now+lease WHERE id IN
  (…) AND status='due' AND due_at <= :now` with `FOR UPDATE SKIP LOCKED` selection on MySQL 8 / PostgreSQL (plain
  conditional UPDATE on SQLite). Only rows the UPDATE changed are dispatched: two ticks never dispatch the same step.
- An expired lease (worker killed) makes the step claimable again by the next tick; the step's idempotency key is
  unchanged, so the action dedupes.
- All writes run in **transaction-doctrine write units**: deadlocks and lock-wait timeouts replay up to 3 times with
  jitter; a final conflict in a request answers **409 + `Retry-After`**, code `CONFLICT_001` (D-20261117-15); in a
  handler it is a Messenger retry. Every "retry" exception implements `LockConflictExceptionInterface`.
- **Optimistic versions** on `automation` (edit, activate, pause) and `enrolment` (pause, unenrol): a stale
  `expectedVersion` answers 409 with the form re-rendered.
- **Unique keys** close the races: `(automation_id, event_key)` (two workers, one event: one run, SC-005),
  `(run_id, step_index)`, `idempotency_key`, `active_key` (two enrol clicks or an action and a person at the same time:
  one active enrolment, the other answers `already_active`).
- Counters use one atomic `UPDATE … WHERE count < :limit`; no read-modify-write.
- Lock order: automation → enrolment → run → run step; never the reverse. `docs/CONCURRENCY.md` lists every write path,
  its lock and its residual risk (as NotifyKit and WorkingTime do).

### 11.4 Actions and idempotency

- **Internal actions** (`EffectClass::Internal`, e.g. `crm.set_field`, `taskboard.create_task`): run in the step's
  write unit when they use the same connection; otherwise in their own write unit followed by the step's (the
  handler's idempotency key makes a retry after a crash between the two harmless: TaskBoard and CrmKit operations accept
  an idempotency key, CrmKit through `CrmApiIdempotencyStoreInterface`).
- **External actions** (`mail`, `external`, `message`): step marked `dispatching` (committed), then the call, then
  `done` / `failed`. A crash in between leaves `dispatching`; the next attempt calls again **with the same key**:
  MailKit dedupes by message key, webhooks carry an `Idempotency-Key` header, NotifyKit by `dedupeKey`. Delivery is
  **at least once with keys** (Q-AK-6); receivers without dedupe may see a duplicate on a crash, documented.
- Retries: `Retry` outcome or a transient exception → `attempts + 1`, `due_at` with backoff, max `max_attempts`; then
  `failed`. `FailedPermanent` → `failed` at once. A `failed` step ends the run (`failure_policy: stop_run`) or skips to
  the next (`continue`). `consecutive_failures` failed runs pause the automation (`circuit_breaker`).
- Actions a step runs may record events: the engine wraps the call in `ChainContext` (per call, cleared at every
  boundary) so those events carry the chain one level deeper, and their messages leave with
  `DispatchAfterCurrentBusStamp` (never on a rolled-back step).

### 11.5 Delays, windows and wait-until

- `delay`: `due_at = now + duration`, then moved forward into the step's `ExecutionWindow` (time zone, weekdays,
  hours, holidays through the WorkingTime bridge when `skip_holidays`). DST-safe (local time arithmetic in the window's
  zone). Delays are stored rows, never Messenger `DelayStamp`s (Q-AK-5): edits, pauses, cancellations and erasure act
  on rows, and no transport limit applies to 365-day delays.
- `wait_until`: condition + `timeout` (≤ `wait_timeout_max`) + `on_timeout` (`continue` | `end`). The step is
  `waiting` with a `<p>automation_wait` row; a `record()` or `signal()` of the same subject wakes it
  (`WakeWaitingStepsMessage`), and the tick re-evaluates waiting steps whose `wait_until_at` passed.
- **Sequences** apply `SequenceSettings.window` to every step, the `daily_cap_per_sender` (atomic counter per sender
  per local day; over the cap the step moves to the next window, never dropped), and **stop rules**: a matching
  `signal()` (reply, meeting booked, unsubscribe; FR-062) stops the enrolment and cancels its pending steps in one write
  unit, before any later step can claim.

### 11.6 FrankenPHP worker safety (REQ-RUNTIME-002, kernel reset false)

- Services are `final readonly` and stateless. Per-call state (after-commit buffer, chain context stack, request
  memo of the tenant) lives in `ExecutionScope` holders cleared at `kernel.request`, `kernel.finish_request`,
  `kernel.terminate`, `WorkerMessageReceived/Handled/FailedEvent` and console boundaries, and implementing
  `ResetInterface` (tested per path, never relied on alone).
- The catalogue (triggers, conditions, actions, recipes) is compiled into the container (immutable).
- Private EM cleared after every write unit and read batch; closed EM detected and reset (transaction-doctrine
  `DoctrineStateRepairer`); no managed entity survives a message or request.
- No tenant, actor or subject data in service properties or statics; HTTP clients per call (OutboundUrlGuard).
- **Tests:** one kernel with `disableReboot()`: 2,000 alternating requests and messages over two tenants (edit, dry
  run, intake, step execution, enrol) assert no crossed rows, no chain context leaking into the next message, memory
  drift ≤ 1 MiB; a Messenger worker loop of 1,000 messages with the same assertions; Igor and `phpstan-frankenphp`
  clean.

## 12. Safety limits, kill switch and audit

### 12.1 Loops and recursion

- **On save / activate:** a cycle check over the tenant's active automations: an edge A → B when an action of A
  `canCause` B's trigger (parameters included, as Beacon's `TaskAutomationCycleDetector`). A cycle refuses activation
  and names the automations along it.
- **At run time:** chain depth ≤ `max_chain_depth` (3) and the visited set (an automation never runs twice in one
  chain) stop loops that appear later (definitions edited behind the check). Stopped runs are logged `stopped`
  (`depth` / `loop`), never silently dropped.
- **Self-triggering by subject:** `runs_per_automation_subject_day` (10) bounds ping-pong through host listeners the
  chain cannot see.
- Enrol actions cannot enrol into the sequence the run belongs to; sequences cannot enrol into each other in a cycle
  (same graph).

### 12.2 Rate limits and caps

| Limit | Default | When reached |
| --- | --- | --- |
| Runs per tenant per hour | 500 | New runs `throttled` (logged) |
| Runs per automation × subject per day | 10 | New runs `skipped` (`subject_cap`) |
| Active automations per tenant / per context | 100 / 25 | Activation refused (422) |
| Steps per automation | 50 | Save refused |
| Mail per sender per local day (sequences) | 200 | Step moved to the next window |
| Webhook calls per tenant per minute | 60 | Step retried later |
| Bulk enrol | 1,000 subjects per request | Refused above; chunked async below |
| UI writes per session | 120 / minute | 429 |

Counters live in `<p>automation_counter` (atomic, shared by all workers) or the host's `cache.rate_limiter` pool for
UI limits; nothing is counted in process memory.

### 12.3 Kill switch

- **Global** (`kill_switch.global`, runtime setting; operator page and `nowo:automation-kit:kill-switch on|off`):
  intake records nothing (events are counted and dropped), ticks claim nothing, in-flight steps finish their current
  attempt. Pending steps stay `due` and resume when switched off (Q-AK-8), with an operator action *Discard pending
  steps older than…*.
- **Per action key** (`kill_switch.actions_disabled`): steps of that action are held.
- **Per tenant** (operator, `operate` permission) and **per automation** (pause).
- Every switch change is audited (§12.4) and shown in a UiKit banner on the automation pages.

### 12.4 Audit trail

- **Run log** (`<p>automation_run` + steps): who (author / run-as), what (automation, revision, step, action key), on
  which subject (id), when, outcome and reason codes: ids only, no content. It is the per-run audit of FR-061.
- **AuditKit bridge** (on when installed): `automation.created`, `.updated`, `.activated` (revision), `.paused`
  (reason), `.archived`, `.dry_run`, `enrolment.manual_enrol` / `manual_unenrol` / `bulk_enrol` (counts),
  `run.cancelled`, `kill_switch.changed`, `limits.changed`. Per-run AuditKit forwarding is off by default (volume,
  Q-AK-10).
- **Events** (after commit, ids only, `AutomationKitEvents::*`): `RunStarted`, `RunFinished` (status, reason),
  `StepFailed` (code), `AutomationPaused` (reason), `EnrolmentStarted`, `EnrolmentEnded` (reason). Never params,
  rendered text or subject values.

## 13. Mail-sending actions (owner decision Q-254-6)

- Any action with `EffectClass::Mail` (built-in `mail.send`, CrmKit's sequence e-mail, host actions) sends **only
  through MailKit**, never through a raw `MailerInterface`. The container refuses to build when a mail action is
  registered and MailKit's `sending_domains` port is missing (until MailKit F-0 ships, the bridge registers no mail
  action).
- **Engine-level guard** (not left to handlers): before every mail step the engine asks MailKit's sending-domain port
  for the tenant and the from-address (`check(tenant, from) → verified | failing(reasons) | unknown`). Only `verified`
  sends. Otherwise the step is `failed` (`sending_domain_unverified`, the reasons in the step's codes), active
  enrolments of that sequence pause (`sending_domain`), and the author gets a notice with the reason (FR-064: "refuses
  the send and tells the sender why").
- **Activation** of an automation with mail steps checks the domain too (refused with the reason), and a daily job
  re-checks the domains of active sequences (FR-064 "daily for active sequences"); a failing domain pauses them.
- **Consent and unsubscribe** belong to the subject's owner (CrmKit): mail actions declare `purpose`
  (`marketing_email`, `service`) and the engine asks the host's `ConsentCheckerInterface` (no default: marketing mail
  actions cannot be activated without one). One-click unsubscribe (RFC 8058) links come from the host's
  `UnsubscribeLinkProviderInterface`; an unsubscribe is a `signal()` that stops the subject's enrolments.
- Daily caps per sender (§12.2), sending windows (§11.5), stop rules (§11.5) apply to every mail step.
- The run log stores the MailKit message id as `result_ref`, never the body or the address.

## 14. Personal data (REQ-DATA-001 … 004)

### 14.1 What is personal

Subject and actor ids (pseudonymous references), run and enrolment rows of a subject (that a person was in a
sequence, when it stopped and why), the author of an automation. Action params are tenant configuration (templates,
field values chosen by the author) and must not contain a third person's personal data (documented, and
`ParamField` kinds steer to references instead of free text).

### 14.2 Erasure and export

- **Subject erasure:** the subject's owner (CrmKit, TaskBoard, host) calls `forgetSubject(tenant, subject)` from its
  own erasure (CrmKit's GDPR hooks, FR-005). It deletes the subject's runs, run steps, waits and enrolments (pending
  steps cancelled first, in one write unit) and keeps only the anonymous daily stats (Q-AK-9).
- **Actor erasure** (UserKit eraser, `getKey(): 'automation_kit'`): the author link of automations becomes NULL and
  those automations pause (`author_no_access`, Beacon spec 216 FR-006); `enrolled_by_*` and `run_as_*` set NULL.
- **Export:** `exportSubject(tenant, subject)` for the subject's owner (sequence names, enrolment dates, status, stop
  reason, steps done with dates; no params); UserKit export provider for actors (automations authored, manual
  enrolments made). Export and erasure cover the same tables (test lists every table).
- **Tenant deletion:** `forgetTenant()` removes every row of the tenant in batches.

### 14.3 Retention

Runs and steps of finished runs: 30 days (`runs_days`, 1–365, Beacon spec 216). Ended enrolments: 90 days. Daily stats:
400 days (no subject). Active runs and enrolments are never purged by age. Purge in keyset batches of 500 via
`PruneMessage`.

## 15. UI contract (UiKit + FormKit, REQ-UI-001/002/003, REQ-TWIG-005)

All pages are opt-in (`web_ui.enabled`), mounted under the host's prefix and layout (UiKit layouts), and embeddable
per context (a board's *Automations* tab renders the list fragment filtered by its `ContextRef`).

| Page | UiKit pattern | Content |
| --- | --- | --- |
| Automations | `ListPage` | Filters (kind, status, context, trigger), server sort, rows per page, row menu (edit, pause / activate, duplicate, archive, run log), phone layout; header actions *New rule*, *New sequence*, *From recipe* |
| Editor | `FormPage` (FormKit) | Name, scope, trigger select + its params, condition tree builder, the **sentence preview** ("When a deal enters *Proposal*, if …, wait 5 days, then create a task and enrol in *Proposal nudge*"), step list (reorder, add, remove). **Each step and each condition is edited in a remote form dialog loaded on demand** (D-20261123-01: 422 inside the dialog, 204 + `X-Nowo-Redirect` on success, a real link as the no-JS fallback); never inline forms per row |
| Activate / dry run | Dialog on demand | Validation result, cycle report, mail domain check; dry run picks ≤ 10 sample subjects (search through `SubjectResolverInterface::search()`) and lists per step what would happen (`Simulation`) |
| Run log | `ListPage` | Filter by automation, status, date, subject; keyset pages; row → run detail (steps timeline with status, codes, attempts, due times); *Cancel run* (CSRF, `operate`) |
| Sequence enrolments | `ListPage` | Per sequence: active / ended, stop reasons, next step due; *Enrol…* (dialog on demand with a subject picker), *Unenrol*; bulk enrol from host lists through `SequencesInterface::enrolMany()` |
| Operator | `SettingsPage` | Kill switches (global, per action, per tenant), throughput and failures per action (7 days), waiting and due backlog, last tick, circuit-breaker pauses; no subject data |

- FormKit for every form (profiles `filter`, `default`); parameter fields generated from `ParamSchema` (§7.5).
- Stimulus controllers (TypeScript): step list reorder, condition builder, dialog opener; compiled assets committed;
  UiKit tokens only; dark mode and reduced motion; axe-clean.
- Translations in 7 locales with key parity (REQ-I18N-002); catalogue labels come from the providers' domains.
- Every template overridable (`REQ-TWIG-001`), documented in `docs/TEMPLATES.md`; `PageContractTestCase` per page.
- Query budgets per page (FR-007): the list runs the same queries with 5 and 100 automations; the run log with 25 rows.

## 16. API and MCP operations

- **Application operations** (`Application/Api/Operation/*`, the CrmKit 0.3 pattern): `ListAutomations`,
  `GetAutomation`, `PauseAutomation`, `ActivateAutomation`, `ListRuns`, `GetRun`, `CancelRun`, `Enrol`, `Unenrol`,
  `ListEnrolments`, `DryRun`. Each checks the tenant and the permission of §9, takes an idempotency key for writes and
  returns bundle VOs. The HTTP controllers and a host's MCP tools are thin layers on top.
- **Public HTTP API** (optional, `api.enabled: false`; canonical `<prefix>/automations/v1/…`, OpenAPI
  `docs/openapi/automations-v1.yaml` with a route-comparison test): `GET automations`, `GET automations/{id}`,
  `POST automations/{id}/pause` · `/activate` (`If-Match: version`), `GET runs`, `GET runs/{id}`, `POST runs/{id}/cancel`,
  `POST sequences/{id}/enrolments` (`Idempotency-Key`), `DELETE enrolments/{id}`, `POST automations/{id}/dry-run`.
  Creating and editing definitions through the API is 1.x (Q-AK-12). Authentication through ApiTokenBundle module
  `automations` (levels `read`, `write`, `operate`) or the host firewall; JSON errors `{"err", "ECODE"}`,
  `Cache-Control: no-store`; no unversioned aliases.
- **MCP:** the bundle ships no MCP server. Beacon (or another host) exposes `automation_*` tools over the operations,
  as it does for `crm_*` (CrmKit 0.3).

## 17. Bridges

| Bridge | Provides | Enabled |
| --- | --- | --- |
| MailKit | `mail.send` action, sending-domain guard (§13) | When MailKit with `sending_domains` is installed |
| NotifyKit | `notify` action; author notices (paused, domain failing) | When installed |
| TaskBoard | Task triggers (spec 216 catalogue, `due_passed` scheduled), `taskboard.create_task`, `.add_comment`, `.move_to_column`, `.add_tag`, `.set_priority`, `.add_member`, `.set_due_relative`; subject resolver `taskboard.task` | When installed |
| Workflow | `workflow.apply_transition` action, `workflow.transition_applied` trigger; subject objects through Workflow's `WorkflowSubjectReferenceResolverInterface` | When installed |
| OutboundUrlGuard | `http.webhook` action (JSON body of declared fields, HMAC-SHA256 signature header, `Idempotency-Key`) | `actions.http_webhook.enabled: true` |
| WorkingTime | Holidays for `skip_holidays` windows | When installed |
| SmsKit | `sms.send` (consent mandatory) | 1.x, `bridges.sms_kit: true` (Q-AK-11) |
| UserKit, AuditKit, ApiToken, InstanceSettings | Erasure / export, audit, API auth, runtime settings | When installed |

CrmKit's triggers, conditions (segment rules) and actions live in **CrmKit's** `Bridge/AutomationKit` (CrmKit roadmap
F-3), depending on this bundle's public SPIs only.

## 18. Migration paths (after 1.0)

- **Beacon spec 216 task automations** (≈ 3,360 lines in `src/Tasks/**/Automation`): rules become automations of
  kind `rule` scoped to `board:<id>` with the TaskBoard bridge; runs keep 30-day retention (no data copy needed beyond
  active rules: rules are copied, run logs restart). Beacon decides the timing (Q-AK-13).
- **CrmKit F-3**: built directly on AutomationKit; no migration.

## 19. Versioning toward 1.0

- 0.x minors per slice (ROADMAP.md); CrmKit F-3 needs S1–S5 (engine, time, sequences, UI, mail).
- Schema changes before 1.0 are listed in `UPGRADING.md` with reference DDL (no migrations shipped; `nowo:automation-kit:schema-check`).
- **1.0.0** when every slice is done, `bundle-req-audit` reports 100 % of the applicable REQs, SEC-004 audits pass
  and the owner approves. After 1.0: additive minors only; public SPIs (§6, §7) follow semver, and new catalogue keys
  are additive.
- Audits: S1 (access, tenancy, idempotency) and S5 (mail and webhooks: automation sending mail, spec 254 *Immediate
  audit*) get an **immediate** SEC-004 audit before their tag; the rest is audited in the batch before 1.0.0.

## 20. Test strategy (coverage gate ≥ 99 % PHP, ≥ 90 % TS)

- **Idempotency (SC-005):** one event delivered to two workers concurrently → one run; a step retried after a crash
  in `dispatching` → the handler sees the same key; two ticks → one dispatch per step.
- **Loops:** cycle refused on activate; a loop created behind the check stops at the configured depth; visited set.
- **Concurrency:** stale version → 409; double enrol → one active enrolment; pause while a step is claimed → cancelled
  at step start; unsubscribe signal racing a mail step → the mail step never sends after the stop commit.
- **Delays and windows:** reference table of 20 cases (weekends, holidays, DST in Europe/Madrid and America/New_York,
  cap overflow to the next window).
- **Access:** author lost access → denied + pause + notice; cross-tenant event → `subject_gone`; another tenant's
  id → 404; forged unsigned message refused.
- **Mail:** unverified / failing / unknown domain refuses with reasons; activation refused; daily re-check pauses;
  no mail action without MailKit `sending_domains` (container test).
- **GDPR:** export ↔ erasure parity; subject erasure cancels pending steps; actor erasure pauses.
- **Collation (MySQL):** `usera` vs `UserA` subjects never cross.
- **Both host kinds**, **worker mode** (§11.6), **UI** (page contracts, CSRF, no-JS fallback, axe), **dry run**
  (write spy: zero writes, zero external calls), **query budgets**.
- **Demo:** Symfony 8.1 + FrankenPHP worker + MySQL + Messenger; e2e with WebP screenshots (REQ-DEMO-013) of the
  list, the editor with a step dialog, a dry run, the run log and enrolments.

## 21. Owner questions (Q-AK)

All seventeen were answered by the owner on 2026-11-23: every recommendation accepted (§22). The options stay below as
history.

| # | Question | Options (kept as history) | Recommendation → decision |
| --- | --- | --- | --- |
| Q-AK-1 | Where does the sequence engine live? CrmKit's roadmap F-3 lists `SequenceStep`, `SendingWindow`, `DailyCap`, `EnrolmentStopReason` as CrmKit VOs | (a) generic enrolments, timed steps, windows, caps and stop rules in AutomationKit; CrmKit ships the mailbox e-mail action, stop signals and its UI entry points; (b) sequences entirely in CrmKit, AutomationKit only rules | Decision 2026-11-23: **(a)**: TaskBoard, Uptime and target projects also need "N timed steps per record"; CrmKit F-3 then names the AutomationKit VOs instead of its own (a CrmKit roadmap edit) |
| Q-AK-2 | Step model in 1.0 | (a) ordered step list with `gate` (if not → end / continue / jump to a later label) and `wait_until`; (b) if / else branches with nested lists; (c) free graph with parallel paths | Decision 2026-11-23: **(a)**: covers FR-060 – FR-062 and the scenario of spec 254, keeps the cycle check and the UI simple; (b) as a 1.x minor if a consumer needs it |
| Q-AK-3 | Condition language | (a) structured tree (`all` / `any` / `not` + registered evaluators), built by the UI; (b) extract WorkflowBundle's closed `GuardExpression` DSL to a small shared package and use it for text conditions; (c) Symfony ExpressionLanguage | Decision 2026-11-23: **(a)** in 1.0 (UI-built, maps to CrmKit segment rules, no parser to audit); (b) later if text conditions are asked for. Not (c): too open (function calls) |
| Q-AK-4 | Who does an automation act as? | (a) its author (last person to activate it), re-checked before every action (Beacon spec 216); (b) also allow a tenant "system" actor with a dedicated permission set | Decision 2026-11-23: **(a)** only in 1.0: every change has a responsible person and loses power with them; (b) can be added later behind `operate` |
| Q-AK-5 | Delay mechanism | (a) stored due rows + a 1-minute scheduler tick; (b) Messenger `DelayStamp` | Decision 2026-11-23: **(a)**: survives edits, pauses, erasure and transports without long delays; precision ±1 minute |
| Q-AK-6 | Delivery guarantee for external actions (mail, webhook, notify) | (a) at least once with a stable idempotency key per step; (b) at most once (mark done before calling, never retry) | Decision 2026-11-23: **(a)**: MailKit, NotifyKit and webhook receivers dedupe on the key; (b) loses mail on a transient failure |
| Q-AK-7 | Retention | (a) runs 30 days, ended enrolments 90 days, anonymous daily stats 400 days; (b) runs 90 days | Decision 2026-11-23: **(a)** (Beacon spec 216 runs: 30 days), configurable 1–365 |
| Q-AK-8 | Global kill switch and pending steps | (a) hold: steps stay due and resume when switched off, with an operator action to discard older ones; (b) cancel every pending step | Decision 2026-11-23: **(a)**: an operator stopping a bad deploy should not lose a week of sequence progress; discarding is explicit |
| Q-AK-9 | Subject erasure | (a) delete runs, steps and enrolments of the subject, keep anonymous daily counts; (b) anonymise the subject id and keep the rows | Decision 2026-11-23: **(a)**: nothing left to link; the counts keep the reports |
| Q-AK-10 | "Every run audited" (FR-061) | (a) the run log is the per-run audit; AuditKit gets definition changes, manual enrolments, cancellations and kill switches; per-run forwarding opt-in; (b) one AuditKit entry per run | Decision 2026-11-23: **(a)**: AuditKit volume stays sane at 500 runs / hour / tenant |
| Q-AK-11 | SMS action (SmsKit) | (a) 1.x bridge after 1.0, off by default, consent port mandatory; (b) in 1.0 | Decision 2026-11-23: **(a)**: spec 254 keeps SMS from records out of scope v1; no consumer yet |
| Q-AK-12 | Public API scope in 1.0 | (a) read, pause / activate, cancel, enrol / unenrol, dry run; definitions edited in the UI only; (b) also create / edit definitions | Decision 2026-11-23: **(a)**: definition editing through the API needs the param-schema contract to be stable first; MCP tools follow the same scope |
| Q-AK-13 | Migrate Beacon's spec 216 task automations to AutomationKit + the TaskBoard bridge | (a) yes, after 1.0, Beacon picks the release; (b) no, keep two engines | Decision 2026-11-23: **(a)**: one engine, one run log, one kill switch; spec 216 is the proof that the engine is generic |
| Q-AK-14 | Built-in `http.webhook` action | (a) yes, off by default, OutboundUrlGuard only, HMAC signature with a per-tenant secret from the host's secret port; (b) no webhooks in 1.0 | Decision 2026-11-23: **(a)**: FR-060 lists "call a webhook through OutboundUrlGuard" |
| Q-AK-15 | Business-day windows and holidays | (a) weekdays + hours + time zone built in, holidays through the optional WorkingTime bridge; (b) weekdays and hours only | Decision 2026-11-23: **(a)** |
| Q-AK-16 | Default limits | Chain depth 3, 500 runs / tenant / hour, 10 runs / automation × subject / day, 100 / 25 active automations per tenant / context, 50 steps, 200 mails / sender / day, bulk enrol 1,000 | Decision 2026-11-23: **Accept** (Beacon spec 216 values where they exist), all configurable and runtime settings |
| Q-AK-17 | First tag | (a) 0.1.0 after S1 (engine core, after its immediate audit) so CrmKit can build F-3 triggers early; (b) first tag only after S5 | Decision 2026-11-23: **(a)**; Packagist registration by the owner after S0, before the tag |

## 22. Answers

**2026-11-23, owner:** Q-AK-1 … Q-AK-17 accepted as recommended:

| # | Decision |
| --- | --- |
| Q-AK-1 | (a) generic sequence engine in AutomationKit; CrmKit ships the mailbox e-mail action, stop signals and UI entry points (CrmKit F-3 roadmap to reference AutomationKit's VOs) |
| Q-AK-2 | (a) ordered step list with `gate` and `wait_until` in 1.0; if / else branches a possible 1.x minor |
| Q-AK-3 | (a) structured condition tree with registered evaluators |
| Q-AK-4 | (a) automations act as their author, re-checked before every action |
| Q-AK-5 | (a) stored due rows + 1-minute scheduler tick |
| Q-AK-6 | (a) at least once with a stable idempotency key per step |
| Q-AK-7 | (a) runs 30 days, ended enrolments 90 days, anonymous daily stats 400 days (configurable 1–365) |
| Q-AK-8 | (a) global kill switch holds pending steps; explicit operator discard |
| Q-AK-9 | (a) subject erasure deletes runs, steps and enrolments; anonymous daily counts kept |
| Q-AK-10 | (a) run log is the per-run audit; AuditKit for definition changes, manual operations and kill switches; per-run forwarding opt-in |
| Q-AK-11 | (a) SmsKit `sms.send` as a 1.x bridge, off by default, consent port mandatory |
| Q-AK-12 | (a) public API 1.0: read, pause / activate, cancel, enrol / unenrol, dry run; no definition editing |
| Q-AK-13 | (a) migrate Beacon spec 216 task automations after 1.0; Beacon picks the release |
| Q-AK-14 | (a) built-in `http.webhook`, off by default, OutboundUrlGuard only, HMAC with a per-tenant secret from the host's secret port |
| Q-AK-15 | (a) weekdays + hours + time zone built in; holidays through the optional WorkingTime bridge |
| Q-AK-16 | Default limits accepted (all configurable runtime settings) |
| Q-AK-17 | (a) first tag 0.1.0 after S1 (after its immediate audit); Packagist registration by the owner after S0, before that tag |
