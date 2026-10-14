# Test prompts

Every prompt and request used to test the Lumen brokers during the V1 → V2 → harness work (2026-10-13 / 14), with what each one checks, what to expect, and what was observed.

Send A2A requests as JSON-RPC `SendMessage` with header `A2A-Version: 1.0` (see README §8 for the full `curl`). For a reply on the same conversation, include the `taskId` and `contextId` from the previous response.

| Broker | Endpoint |
|---|---|
| V2 single orchestrator | `https://<your-ingress-gateway-url>/lumen_broker/` |
| V2 harness | `https://<your-ingress-gateway-url>/lumen_broker_harness/` |

---

## 1. Broker prompts

### P1 — Invoice (short)

```text
can you check my current invoice?
```

| | |
|---|---|
| Checks | Trigger → orchestrator → Billing Agent → echo works end to end; Gemini connection and A2A egress reachable |
| Expected | `TASK_STATE_COMPLETED`; invoice $18,400 vs typical $14,200, $2,800 bursting + $1,400 upgrade, $500 goodwill credit |
| Used on | V2 single orchestrator 1.0.0 (first smoke test after deploy) |
| Observed | Completed in ~40 s with the expected figures |

### P2 — Invoice only

```text
Hello I work at Hartwell, can you check my current invoice?
```

| | |
|---|---|
| Checks | Classifier sets only `wants_invoice`; only the Invoice section appears; the goodwill credit stays under Invoice; Gemini connection still works after API Manager changes |
| Expected | `TASK_STATE_COMPLETED`; one **Invoice** section with amounts and the credit; no Recommendations, Diagnostics or Appointment sections |
| Observed | Harness 1.0.6: completed in 11 s, but the credit was under "Recommendations" (bug). 1.0.7–1.0.9: 7/7 runs (3 + 3 + 1) with only an Invoice section, credit inside it |

### P3 — Invoice + recommendations

```text
Hello I work at Hartwell, can you check my current invoice and recommend improvements to our network?
```

| | |
|---|---|
| Checks | Both read-only branches run (`getInvoice`, `getRecommendations`); each result lands in its own section; no unexpected question to the customer |
| Expected | `TASK_STATE_COMPLETED`; **Invoice** (amounts + credit) and **Recommendations** (current setup: Lumen DIA and SD-WAN across 12 locations, 43% traffic growth, recommended solutions and cost impact) |
| Observed | 1.0.7: reply was only a progress note (bug, README §7.13). 1.0.8: 1/2 correct, the other asked "Would you like to proceed with just the invoice details?" (bug, §7.14). 1.0.9: 4/4 correct |

### P4 — Full request (the main scenario)

```text
Hello I work at Hartwell, can you check my current invoice? Also, we encountered downtime on our network at 2500 Fairmount St, Dallas area. I need a field engineer appointment for a DIA circuit on-site inspection. Only include certified fiber/DIA technicians.
```

| | |
|---|---|
| Checks | Downtime always triggers diagnostics; invoice retrieved; slots offered; the broker pauses for the customer's choice and the question contains everything needed to answer it |
| Expected | `TASK_STATE_INPUT_REQUIRED`; one message with the invoice, diagnostics (degraded DIA circuit, fiber node fault on a regional segment, failover failed, on-site visit required) and slots **A** 9:00–11:00 AM, **B** 11:00 AM–1:00 PM, **C** 1:00–3:00 PM CT |
| Observed | See the table below |

| Broker / version | Runs | Result |
|---|---|---|
| Single orchestrator 1.0.0 | 1 (sent from an A2A client) | Paused, but the question was empty ("Please review them…"), no invoice or slots (README §7.5) |
| Single orchestrator 1.0.1 | 2 | Paused with invoice and slots; diagnostics included in one run, skipped in the other |
| Harness 1.0.3 | 1 | Failed in 4 s: "validation error" (inline enums, README §7.1) |
| Harness 1.0.4–1.0.5 | 2 | Paused with invoice, diagnostics and slots |
| Harness 1.0.5 | 1 | Paused after diagnostics only: "Would you like to proceed with checking your current invoice…?" (README §7.4) |
| Harness 1.0.6 | 5 | 5/5 paused with diagnostics and slots; 4/5 with the invoice (run 4 returned the GRACEFUL FAILURE line instead) |
| Harness 1.0.8 | 2 | 2/2 paused with invoice, diagnostics and slots |
| Harness 1.0.9 | 1 | Paused with invoice, diagnostics and slots |

### P5 — Slot choice (turn 2 of P4)

```text
B
```

Sent on the same `taskId` / `contextId` as a P4 response that ended in `TASK_STATE_INPUT_REQUIRED`.

| | |
|---|---|
| Checks | The graph resumes at `chooseSlot`; `bookingRouter` lets the booking through only for A/B/C; `bookSlot` books exactly the chosen slot; earlier results carry through to the final reply |
| Expected | `TASK_STATE_COMPLETED`; sections Invoice, Network diagnostics, Field service appointment; slot B, 11:00 AM–1:00 PM CT, `SR-20240314-DAL-0893` |
| Observed | Harness 1.0.4: failed, `reasoning_iterations` limit (bookSlot cap 2, README §7.6). 1.0.5, 1.0.6, 1.0.9: completed, booked slot B, `SR-20240314-DAL-0893` |

Same-task replay: from an A2A client, P4 was sent a second time on a task that was already waiting for input. The broker treated it as the answer to its question and returned the full invoice + slots question again (single orchestrator 1.0.1). That's expected A2A behaviour, not a bug.

---

## 2. Component checks (no LLM)

These check the pieces the broker depends on, directly.

### C1 — MCP server tools

```bash
U=https://<field-service-mcp-server-url>/mcp
H=(-H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream')
curl -s "${H[@]}" "$U" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

| Expected | Observed |
|---|---|
| Two tools: `get-field-service-slots` (no inputs) and `confirm-field-service-booking` (required `slotId`) | As expected after the MCP server change on 2026-10-14. Before it, both tools had no inputs |

### C2 — Booking per slot

```bash
curl -s "${H[@]}" "$U" -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"confirm-field-service-booking","arguments":{"slotId":"A"}}}'
```

| `slotId` | Expected / observed |
|---|---|
| `"A"` | `SR-20240314-DAL-0892`, 09:00–11:00 AM CT |
| `"B"` | `SR-20240314-DAL-0893`, 11:00 AM–01:00 PM CT |
| `"C"` | `SR-20240314-DAL-0894`, 01:00–03:00 PM CT |
| missing | Rejected: `slotId … Required` |
| `"D"` | `No matching scenario found for input` |

### C3 — A2A agent cards

```bash
curl -s https://<billing-agent-url>/.well-known/agent-card.json
curl -s https://<network-diagnostic-agent-url>/.well-known/agent-card.json
curl -s https://<solution-advisor-agent-url>/.well-known/agent-card.json
curl -s https://<your-ingress-gateway-url>/lumen_broker_harness/.well-known/agent-card.json
```

| Expected | Observed |
|---|---|
| HTTP 200; A2A v0.3 cards with one skill each (`get_invoice_analysis`, `diagnose-network-circuits`, `recommend-network-upgrades`); the broker card lists 5 skills | As expected. Two agent cards contain raw control characters in their descriptions and need lenient JSON parsing |

---

## 3. Regression set

After any change to the graph or prompts, run at least:

| # | Prompt | Runs | Pass when |
|---|---|---|---|
| 1 | P2 invoice only | 3 | Completed; only an Invoice section; credit inside it |
| 2 | P3 invoice + recommendations | 3 | Completed; Invoice + Recommendations, each with real content; no question to the customer |
| 3 | P4 full request | 5 | Input required; invoice, diagnostics and slots A/B/C all in the question |
| 4 | P4 then P5 "B" | 1 | Completed; slot B, `SR-20240314-DAL-0893` |

Also send "none" as the P5 reply at least once: it should complete without a booking. That case hasn't been tested yet.

The mock agents and the A2D mock server return fixed data, so the figures above are the same on every run; only the wording changes.
