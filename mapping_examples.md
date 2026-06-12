# ISP Recharge Ticket Migration — 5 Cases, 1 Real Sample Each, Old→New Column Mapping

Data as of 2026-06-12 (IST). Old source = Snowflake via Metabase (live Fivetran replicas, `_fivetran_deleted=false`).
New schemas read from `csp-os-yaml/services/` migration SQL + entities (file:line in repo).

**Join key across old tables = NAS_ID** (`isp_expiry_tracker.nas_id` = `recharge_tickets.nas_id` = `recharge_requests.nas_id` = `t_device.nasid` = `tasks.extra_data:nas_id` for ROUTER_PICKUP).

**Legend for the "Source" column:**
- `OLD: table.column` — direct value exists in old source (sample value shown)
- `CONST: x` — fixed constant / schema default
- `CASE` — pinned by your 5-case definition
- `MINT` — new identifier created at migration time (no old equivalent)
- `❓ DERIVE` — no old column holds this; awaiting your derivation logic (candidates noted, NOT inferred)

---

## The 5 chosen samples

| # | Target state | NAS_ID | Why it classified here |
|---|---|---|---|
| 1 | ACTIVE + candidate EXISTS | 281474977397447 | No open pickup; expiry 2027-01-15 > today; PENDING ticket 335103 |
| 2 | ACTIVE + candidate DOES NOT EXIST | 281474977310435 | No open pickup; expiry 2027-02-25 > today; no PENDING ticket |
| 3 | PAUSED + candidate EXISTS | 281474977397713 | No open pickup; expiry 2026-06-11 < today; PENDING ticket 591036 |
| 4 | PAUSED + candidate DOES NOT EXIST | 281474977394766 | No open pickup; expiry 2026-06-11 < today; no PENDING ticket |
| 5 | PENDING_DEACTIVATION (no candidate) | 281474977218548 | Open ROUTER_PICKUP task (status=1) created 2026-06-07; latest plan_start 2026-05-16 is NOT > pickup creation → case 5 (note: its expiry 2026-08-31 is in future — pickup branch wins) |

Old-key identity per sample:

| | Case 1 | Case 2 | Case 3 | Case 4 | Case 5 |
|---|---|---|---|---|---|
| NAS_ID | 281474977397447 | 281474977310435 | 281474977397713 | 281474977394766 | 281474977218548 |
| PARTNER_ID (= t_device.lco_account_id) | 281749855022004 | 281749854635402 | 281749854733695 | 281749854659753 | 281749854641551 |
| CUSTOMER_ACCOUNT_ID (= t_device.user_account_id) | 281749855023763 | 281749854838240 | 281749855023992 | 281749855021517 | 281749854761328 |
| DEVICE_ID | GX148844 | GX114667 | SY099824 | SY080218 | SY067775 |
| expiry_tracker row | 104002 | 62234 | 48171 | 38909 | 183342 |
| PENDING recharge_ticket | 335103 | — | 591036 | — | — |
| recharge_request (latest/linked) | 565475 | 831970 | 868872 | 879878 | 753991 |
| Open ROUTER_PICKUP task | — | — | — | — | 1780814020607791 |
| t_device.status | INSTALLED | INSTALLED | INSTALLED | INSTALLED | TO_BE_PICKED_UP |

---

## TABLE 1 — `connections` (CLOS) — row seeded in ALL 5 cases

| New column | Source | Case 1 | Case 2 | Case 3 | Case 4 | Case 5 |
|---|---|---|---|---|---|---|
| connection_id | MINT | (new id) | (new id) | (new id) | (new id) | (new id) |
| customer_id | OLD: recharge_requests.CUSTOMER_ACCOUNT_ID (= t_device.USER_ACCOUNT_ID) — final ID scheme = ❓ | 281749855023763 | 281749854838240 | 281749855023992 | 281749855021517 | 281749854761328 |
| csp_id | OLD: PARTNER_ID — but new col is **UUID** → needs partner→CSP id-map ❓ | 281749855022004 | 281749854635402 | 281749854733695 | 281749854659753 | 281749854641551 |
| zone_id | ❓ DERIVE (no old col in scoped tables) | — | — | — | — | — |
| service_address (JSONB) | OLD: tasks.extra_data:address — exists ONLY when pickup task exists; else ❓ | ❓ | ❓ | ❓ | ❓ | {"address":"B98, Ch Dham Singh Marg, Dayalpur, New Mustafabad, New Delhi","pincode":"110090","city":"Delhi","lat":28.7212174,"lng":77.2732254} |
| plan_id | OLD: isp_expiry_tracker.PLAN_ID — old plan id→new plan catalogue map = ❓ | 1 | 1 | 2 | 1 | 2 |
| current_state | CASE | ACTIVE | ACTIVE | PAUSED | PAUSED | PENDING_DEACTIVATION |
| state_timestamp | ❓ DERIVE (candidates: migration ts / tracker.UPDATED_AT / pickup CREATED for case 5) | — | — | — | — | — |
| pause_reason | ❓ DERIVE (cases 3,4 only; NULL for 1,2,5?) | NULL | NULL | ❓ | ❓ | NULL |
| pause_count | ❓ DERIVE (schema default 0) | — | — | — | — | — |
| p76_timer_start | ❓ DERIVE (case 5 candidate: tasks.CREATED = 2026-06-07T11:51:41+05:30) | NULL | NULL | ❓ | ❓ | ❓ |
| retry_count | CONST: 0 | 0 | 0 | 0 | 0 | 0 |
| retry_exhausted | CONST: FALSE | FALSE | FALSE | FALSE | FALSE | FALSE |
| installation_complete_at | ❓ DERIVE (t_device has no install ts in scoped cols; STATUS_UPDATED_AT is last status change only) | — | — | — | — | — |
| activated_at | ❓ DERIVE | — | — | — | — | — |
| deactivated_at | CONST: NULL (none of the 5 states is DEACTIVATED) | NULL | NULL | NULL | NULL | NULL |
| total_active_duration | ❓ DERIVE (default 0) | — | — | — | — | — |
| total_pause_duration | ❓ DERIVE (default 0) | — | — | — | — | — |
| version | CONST: 1 | 1 | 1 | 1 | 1 | 1 |
| correlation_id / causation_id | ❓ DERIVE (migration marker?) | — | — | — | — | — |
| deactivation_reason | ❓ DERIVE (case 5 candidate: tasks.REMARKS = "Disconnect / Discontinue service") | NULL | NULL | NULL | NULL | ❓ |
| prior_caeo_state | ❓ DERIVE (enum: CAEO_STATE_ENABLED / CAEO_STATE_BLOCKED / CAEO_STATE_BLOCKED_BY_SUPPLY / CLOSED) | — | — | — | — | — |
| latest_recharge_window_end | OLD: isp_expiry_tracker.EXPIRY_DATE | 2027-01-15T12:06:00+05:30 | 2027-02-25T23:59:59+05:30 | 2026-06-11T23:59:59+05:30 | 2026-06-11T23:59:59+05:30 | 2026-08-31T00:00:00+05:30 |
| created_at / updated_at | ❓ DERIVE (migration ts vs tracker.CREATED_AT) | — | — | — | — | — |

---

## TABLE 2 — `customer_access_states` (CAEOS) — row seeded in ALL 5 cases

| New column | Source | Case 1 | Case 2 | Case 3 | Case 4 | Case 5 |
|---|---|---|---|---|---|---|
| connection_id | MINT (same id as connections row) | (new id) | (new id) | (new id) | (new id) | (new id) |
| customer_id | OLD: recharge_requests.CUSTOMER_ACCOUNT_ID | 281749855023763 | 281749854838240 | 281749855023992 | 281749855021517 | 281749854761328 |
| caeo_state | ❓ DERIVE per case (enum: CAEO_STATE_ENABLED / CAEO_STATE_BLOCKED / CAEO_STATE_BLOCKED_BY_SUPPLY / CLOSED) | ❓ | ❓ | ❓ | ❓ | ❓ |
| entitlement_end | OLD: isp_expiry_tracker.EXPIRY_DATE | 2027-01-15T12:06:00+05:30 | 2027-02-25T23:59:59+05:30 | 2026-06-11T23:59:59+05:30 | 2026-06-11T23:59:59+05:30 | 2026-08-31T00:00:00+05:30 |
| supply_disruption_start | ❓ DERIVE (likely NULL — no old signal) | — | — | — | — | — |
| cumulative_supply_disruption | CONST: 0 (default) | 0 | 0 | 0 | 0 | 0 |
| prior_access_state | ❓ DERIVE | — | — | — | — | — |
| payment_reference_id | OLD candidate: recharge_requests.CUSTOMER_REFERENCE_ID — confirm ❓ | custGen_9911296554_197536343_0 | custGen_9560405262_202554339_0 | custGen_7065128087_203169299_0 | OUTAGE_PLAN1781075401111 | custGen_9891671980_201074960_1 |
| closed_at | CONST: NULL | NULL | NULL | NULL | NULL | NULL |
| version | CONST: 1 | 1 | 1 | 1 | 1 | 1 |
| correlation_id / causation_id | ❓ DERIVE | — | — | — | — | — |
| cl_state | CASE (mirrors connections.current_state) | ACTIVE | ACTIVE | PAUSED | PAUSED | PENDING_DEACTIVATION |
| created_at / updated_at | ❓ DERIVE | — | — | — | — | — |

---

## TABLE 3 — `recharge_gates` (RV) — seeding scope per case = your call (presumably all 5; gate is per latest confirmed recharge)

| New column | Source | Case 1 | Case 2 | Case 3 | Case 4 | Case 5 |
|---|---|---|---|---|---|---|
| id | MINT (gen_random_uuid) | (uuid) | (uuid) | (uuid) | (uuid) | (uuid) |
| connection_id | MINT (same as connections) | (new id) | (new id) | (new id) | (new id) | (new id) |
| csp_id | OLD: PARTNER_ID (TEXT col here — old id usable pending final id scheme ❓) | 281749855022004 | 281749854635402 | 281749854733695 | 281749854659753 | 281749854641551 |
| detection_source | CONST: 'MIGRATION' (enum value exists exactly for this: CSP / TELEMETRY / MIGRATION) | MIGRATION | MIGRATION | MIGRATION | MIGRATION | MIGRATION |
| recharge_reference_id | ❓ AMBIGUOUS — candidates: recharge_requests.CUSTOMER_REFERENCE_ID vs recharge_tickets.TRANSACTION_ID | custGen_9911296554_197536343_0 / ISP-281749855022004-20260405T071317Z | custGen_9560405262_202554339_0 | custGen_7065128087_203169299_0 / ISP-281749854733695-...-20260609T115534Z | OUTAGE_PLAN1781075401111 | custGen_9891671980_201074960_1 |
| window_start | OLD candidate: recharge_requests.CUSTOMER_PLAN_START — confirm ❓ | 2027-01-14T05:30:00+05:30 | 2026-06-02T14:36:05+05:30 | 2026-06-09T17:25:32+05:30 | 2026-06-27T17:00:00+05:30 ⚠ future (OUTAGE_PLAN comp) | 2026-05-16T11:39:34+05:30 |
| window_end | ❓ AMBIGUOUS — isp_expiry_tracker.EXPIRY_DATE vs recharge_requests.CUSTOMER_PLAN_EXPIRY (they DISAGREE in case 1: 2027-01-15 vs 2027-02-11) | 2027-01-15T12:06+05:30 / 2027-02-11T05:30+05:30 | 2027-02-25T23:59:59 / 2026-06-30T14:36 ⚠ disagree | 2026-06-11T23:59:59 / 2026-09-01T17:25 ⚠ disagree | 2026-06-11T23:59:59 / 2026-06-29T17:00 ⚠ disagree | 2026-08-31T00:00 / 2026-05-23T11:39 ⚠ disagree |
| recharge_period_days | ❓ DERIVE (default 30; datediff of chosen window?) | — | — | — | — | — |
| confirmation_timestamp | OLD candidate: recharge_requests.CREATED_AT — confirm ❓ | 2026-04-05T12:43:17+05:30 | 2026-06-02T14:36:05+05:30 | 2026-06-09T17:25:34+05:30 | 2026-06-10T12:40:02+05:30 | 2026-05-16T11:39:34+05:30 |
| approaching_emitted / expired_emitted | ❓ DERIVE (booleans; presumably f(window_end vs today) — your logic) | ❓ | ❓ | ❓ | ❓ | ❓ |
| created_at / updated_at | ❓ DERIVE (migration ts) | — | — | — | — | — |

---

## TABLE 4 — `supply_recharge_obligations` (CAEOS) — row seeded ONLY where candidate exists (cases 1 & 3); no row for 2, 4, 5

| New column | Source | Case 1 | Case 3 |
|---|---|---|---|
| obligation_ref | MINT (must equal TAS candidate's authority_entity_id) | (new ref) | (new ref) |
| connection_id | MINT (same as connections) | (new id) | (new id) |
| csp_id | OLD: PARTNER_ID | 281749855022004 | 281749854733695 |
| window_end | ❓ DERIVE — candidates: recharge_tickets.EXPIRY_DATE (2027-02-13 / 2026-09-10) vs tracker.EXPIRY_DATE | ❓ | ❓ |
| reason | ❓ DERIVE per case (enum: PROACTIVE / REACTIVE only) | ❓ | ❓ |
| status | CONST: 'OPEN' (only ACTION_REQUIRED migrates; note: unique index allows ONE OPEN obligation per connection) | OPEN | OPEN |
| customer_entitlement_end | OLD: isp_expiry_tracker.EXPIRY_DATE (NOT NULL col) | 2027-01-15T12:06:00+05:30 | 2026-06-11T23:59:59+05:30 |
| created_at | OLD candidate: recharge_tickets.CREATED_AT — confirm ❓ | 2026-04-05T12:43:17+05:30 | 2026-06-09T17:25:34+05:30 |
| updated_at | ❓ DERIVE | — | — |
| resolved_at | CONST: NULL | NULL | NULL |

---

## TABLE 5 — `recharge_execution_candidates` (TAS) — row seeded ONLY in cases 1 & 3, state = ACTION_REQUIRED

The old `recharge_tickets` table already carries 4 same-named new-system columns (EXECUTION_ID, FLOW_TYPE, RENEWAL_EXECUTION_STATUS, RENEWAL_EXECUTION_SUB_STATUS) — direct copies.

| New column | Source | Case 1 (ticket 335103) | Case 3 (ticket 591036) |
|---|---|---|---|
| execution_candidate_id | MINT (uuid) | (uuid) | (uuid) |
| state_version | CONST: 0 (default) | 0 | 0 |
| authority_entity_id | MINT: = supply_recharge_obligations.obligation_ref | (new ref) | (new ref) |
| csp_id | OLD: recharge_tickets.PARTNER_ID | 281749855022004 | 281749854733695 |
| connection_id | MINT (same as connections) | (new id) | (new id) |
| customer_id | OLD: recharge_requests.CUSTOMER_ACCOUNT_ID | 281749855023763 | 281749855023992 |
| candidate_type | CONST: 'RECHARGE_OBLIGATION' | RECHARGE_OBLIGATION | RECHARGE_OBLIGATION |
| task_family | CONST: 'RECHARGE' | RECHARGE | RECHARGE |
| aggregation_type | CONST: 'BATCH' | BATCH | BATCH |
| state | CASE: ACTION_REQUIRED | ACTION_REQUIRED | ACTION_REQUIRED |
| reason_code | ❓ DERIVE per case (enum incl. SUPPLY_RECHARGE_NEEDED_PROACTIVE / SUPPLY_RECHARGE_NEEDED_REACTIVE / URGENT_WINDOW_APPROACHING / OVERDUE_WINDOW_PASSED …) | ❓ | ❓ |
| reason_display_template | ❓ DERIVE (NOT NULL) | ❓ | ❓ |
| deadline_at | ❓ DERIVE (no old col) | — | — |
| obligation_window_start | OLD candidate: recharge_tickets.START_DATE — confirm ❓ | 2027-01-15T00:00:00+05:30 | 2026-08-12T00:00:00+05:30 |
| obligation_window_end | OLD candidate: recharge_tickets.EXPIRY_DATE — confirm ❓ | 2027-02-13T23:59:59+05:30 | 2026-09-10T23:59:59+05:30 |
| object_id | ❓ DERIVE (NOT NULL — what object does it point at?) | ❓ | ❓ |
| locality | ❓ DERIVE (NOT NULL; no locality in old recharge tables — t_device has only lat/lng) | ❓ | ❓ |
| batch_key | ❓ DERIVE (NOT NULL, batching pattern) | ❓ | ❓ |
| incident_key | CONST: NULL | NULL | NULL |
| is_csp_actionable | CONST: TRUE (default) | TRUE | TRUE |
| threshold_state | ❓ DERIVE per case (enum: NORMAL / URGENT / OVERDUE) | ❓ | ❓ |
| latest_attention_at | OLD candidate: recharge_tickets.CREATED_AT (NOT NULL) — confirm ❓ | 2026-04-05T12:43:17+05:30 | 2026-06-09T17:25:34+05:30 |
| latest_attention_version | CONST: 1 (default) | 1 | 1 |
| latest_attention_type | ❓ DERIVE (enum: STATE_CHANGE / THRESHOLD_CROSSING / STRUCTURAL_REOPEN / OPERATIONAL; NOT NULL) | ❓ | ❓ |
| latest_attention_reason | = reason_code (❓) | ❓ | ❓ |
| acked_attention_version | CONST: 0 (default) | 0 | 0 |
| commission_status | OLD: recharge_tickets.INCENTIVE_CLAIMED=false → 'UNCLAIMED' (mapping to 6-value enum = confirm ❓) | UNCLAIMED | UNCLAIMED |
| commission_claimed_at | OLD: recharge_tickets.INCENTIVE_CLAIM_DATE | NULL | NULL |
| commission_disbursed_at | CONST: NULL | NULL | NULL |
| resolved_at / cancelled_at / cancellation_context | CONST: NULL (state is ACTION_REQUIRED) | NULL | NULL |
| customer_name | OLD: recharge_tickets.CUSTOMER_NAME | IX Device | Sunny |
| connection_short_id | ❓ DERIVE (derived from minted connection_id?) | ❓ | ❓ |
| wan_type | OLD: recharge_tickets.WAN_TYPE | PPPoE | PPPoE |
| identifier | OLD: recharge_tickets.IDENTIFIER | irshad6554 | sunny_1449 |
| device_id | OLD: recharge_tickets.DEVICE_ID (= t_device.DEVICE_ID) | GX148844 | SY099824 |
| customer_phone | OLD: recharge_tickets.CUSTOMER_PHONE | 9911296554 | 7065128087 |
| speed_limit | OLD: recharge_requests.REQUESTED_SPEED (ticket has no speed col; default 100) | 50 | 100 |
| execution_id | OLD: recharge_tickets.EXECUTION_ID | NULL (empty in old) | NULL (empty in old) |
| flow_type | OLD: recharge_tickets.FLOW_TYPE | NULL (empty in old) | NULL (empty in old) |
| renewal_execution_status | OLD: recharge_tickets.RENEWAL_EXECUTION_STATUS | NULL (empty) | NULL (empty) |
| renewal_execution_sub_status | OLD: recharge_tickets.RENEWAL_EXECUTION_SUB_STATUS | NULL (empty) | NULL (empty) |
| created_at / updated_at | ❓ DERIVE (ticket.CREATED_AT vs migration ts) | — | — |

Note: old `recharge_tickets` columns with NO new-system home in this table: PARTNER_PLAN_ID, FIXED_COMMISSION (₹264.6 / ₹294.0), INCENTIVE_ELIGIBLE, TRANSACTION_ID, RECHARGE_REQUEST_ID — flag for your call (commission amount in particular looks load-bearing).

---

## Findings / caveats surfaced while sampling

1. **T_DEVICE coverage gap**: the first randomly-picked case-2 and case-4 NAS IDs (281474977238483, 281474977218368) have **NO t_device row at all** — partner-own-router ISP customers exist in the migration population without any Wiom device. Any mapping sourced from t_device (device_id, user_account_id) must handle this NULL population. (Samples above were re-picked to have devices for illustration.)
2. **Expiry disagreement**: `isp_expiry_tracker.EXPIRY_DATE` ≠ `recharge_requests.CUSTOMER_PLAN_EXPIRY` in 4 of 5 samples (tracker can be ahead or behind). You must pick the canonical source for `recharge_gates.window_end` / `entitlement_end`. (Tracker is what the old flow's bifurcation uses.)
3. **Case 5 trumps expiry**: sample 5 has future expiry (2026-08-31) but an open pickup with plan_start NOT newer than pickup → PENDING_DEACTIVATION, exactly per your rule. Worth confirming this is intended (active-entitlement customers landing in PENDING_DEACTIVATION).
4. **Old PENDING ticket statuses**: recharge_tickets.STATUS population = CLAIMED 547,535 / INVALID 37,594 / **PENDING 22,774** / SUCCESS 32 / FAILED 11 / IN_PROGRESS 1.
5. **Open pickup volume**: ROUTER_PICKUP status 0 = 4,418, status 1 = 3,016 (statuses 2/3 ignored per your rule).
6. **NOT NULL pressure**: new NOT-NULL columns with no old source that MUST get a derivation before any load: `connections.zone_id`, `service_address` (cases without pickup task), `candidates.reason_code`, `reason_display_template`, `object_id`, `locality`, `batch_key`, `latest_attention_type`, `obligations.reason`, `caeo_state`, `recharge_gates.recharge_reference_id`, `confirmation_timestamp`.
