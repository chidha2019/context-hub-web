# ContextHub — POC Scope & Implementation Plan

---

## 1. The proposition

We have built a working prototype interface that *describes* ContextHub: a unified, governed
context layer that sits between an organization's systems of record and the people and AI agents
that consume them. It has been useful — it has aligned everyone on the vision.

But it describes the thesis rather than demonstrating it.

This POC closes that gap. It takes a single, deliberately narrow slice and makes it real end to
end, so the thesis can be shown working rather than argued for.

> **The thesis under test:** one governed knowledge graph, many persona-scoped views, served to
> humans and AI agents through the same path — with access control, PII handling, provenance and
> audit applied *at retrieval time*, not bolted on afterwards.

The emphasis on *at retrieval time* is the whole argument. Any system can filter what it shows on
screen. The claim we need to prove is that the context itself is scoped before it ever reaches a
model or a person — because that is the only version of the claim that survives an agent calling
the system directly.

---

## 2. What the POC must prove

Five claims, in dependency order. Each phase in Section 7 exists to establish one of them.

| # | Claim | Why it matters |
|---|---|---|
| 1 | Context can be unified across heterogeneous sources without re-authoring it | If we need humans to restructure knowledge first, there is no product |
| 2 | Identity resolves across systems automatically | The same person, service and customer appear under different names everywhere. Without resolution there is no graph, only silos with a shared search box |
| 3 | Access control is enforced on the context, not on the display | Two people asking the same question must receive genuinely different context — this is the differentiator |
| 4 | Answers are grounded and fully traceable | An enterprise cannot adopt what it cannot audit |
| 5 | AI agents consume through the same governed path | If agents get a side door, the governance story is theatre |

Claims 3 and 5 are the ones that distinguish ContextHub from enterprise search. They receive
disproportionate attention in the plan, and they are the two acceptance criteria in Section 8.

---

## 3. Synthetic data

**Synthetic data gives us three things we cannot otherwise have:**

- **Determinism.** The same demo, every time, for every audience.
- **Offline operation.** No dependency on network or third-party availability at review time.
- **Planted defects.** This is the important one.

### The corpus is deliberately flawed

A clean corpus cannot prove that governance does anything. So the generated organization contains,
on purpose:

| Planted defect | What it lets us prove |
|---|---|
| Personal data scattered through informal conversation | Detection and masking actually operate on retrieved context |
| An engineer stating a policy incorrectly, contradicting the authoritative document | Trust scoring ranks the authoritative source above the casual one |
| Attempted prompt-injection payloads | The layer blocks hostile content before it reaches a model |
| One conversation channel classified as confidential | Access control produces a visible, correct denial |
| The same person, service and customer named differently in each source | Cross-system identity resolution is earned, not configured |

Critically, the corpus **never contains the mapping between those different names.** The ingestion
pipeline has to derive it. If we shipped the answer key inside the data, we would be demonstrating
nothing.

### The organization

A fictional B2B payments company of roughly forty people across five teams, with eighteen months of
history. Payments is a good choice of domain: it naturally carries sensitive data, real policy
documents, incident history and clear service ownership — so the governance story has genuine
material to work with rather than being staged.

---

## 4. Input sources — the reasoning

We are ingesting **three** sources: **GitHub**, **Jira** and **Slack**.

### The selection criterion

We did not choose by popularity or by what is easiest to integrate. We chose the **minimum set that
spans the three distinct shapes of enterprise knowledge** — because each shape stresses a different
capability, and a POC that only handles one shape proves very little.

| Shape | Source | What it contributes | What it stresses |
|---|---|---|---|
| **Authoritative & structured** | GitHub | Code, architecture decisions, policy documents, ownership records | Provenance and the top of the trust hierarchy |
| **Procedural & semi-structured** | Jira | Tickets, incidents, status, resolution history, cross-references | Temporal context and linking entities across systems |
| **Tacit & unstructured** | Slack | Conversation threads — where knowledge actually lives | Everything hard: personal data, contradictions, hostile content, confidentiality |

### Why Slack matters most

Slack is the only source where governance is *visibly load-bearing*. It carries the personal data,
the statement that contradicts the official policy, the injection attempts and the confidential
channel. Remove Slack and the POC becomes a competent document-retrieval demo — which is not what
we are claiming to build.

It is also the honest case. Every organization's most valuable context is in conversation, and
every organization's governance anxiety is about exactly that.

### Why not more sources

**Because a fourth source of the same shape adds volume, not proof.** Confluence is
document-shaped, like the documentation already coming from GitHub. Salesforce is record-shaped.
Neither exercises a capability the current three do not already exercise, so neither changes what
the POC demonstrates.

The connector layer is built to a common interface from the start — adding sources is a scaling
exercise, not a research risk. We would rather prove three sources deeply than gesture at eight
shallowly.

---

## 5. Personas — the reasoning

We are demonstrating **three** personas: **Data Engineer**, **Digital Engineer** and
**Citizen Developer**.

### The selection criterion

The criterion is a **privilege gradient**, not a role survey.

The purpose of personas in this POC is not to show that different roles see different dashboards —
that is a presentation concern and we have already shown it in the prototype. The purpose is to
prove that **access scope changes what context exists for you at all.**

That requires personas separated by genuine clearance. Three roles with similar access would
produce three near-identical answers in different colours, which proves nothing and would rightly
be called out as such.

| Persona | Position on the gradient | What it demonstrates |
|---|---|---|
| **Data Engineer** | Broad clearance | The control case — what complete, correctly-sourced context looks like |
| **Digital Engineer** | Mid clearance, agent-facing | Partial withholding, and the path AI agents consume through |
| **Citizen Developer** | Narrow, guard-railed | The contrast case — the layer actively withholding, and saying so |

Two ends and a middle. A survey of every role would demonstrate variety; three points on a gradient
demonstrate **enforcement** — which is the thing under review.

### Why the low-privilege persona is non-negotiable

An early version of this plan proposed only the two engineering personas. Both hold broad access,
which would have made the access-control story invisible — the demo would show the same answer
twice.

The Citizen Developer exists specifically to be told no. That refusal, rendered clearly and with a
reason, is the single most persuasive moment in the demonstration.

---

## 6. How we will demonstrate the capability

Two demonstrations. The first proves governed context; the second proves the agent path.

### Demonstration A — one question, three clearances

The same question is asked three times, changing only who is asking:

> *"Who owns the billing retry logic, and what is the current retry policy?"*

| | **Data Engineer** | **Digital Engineer** | **Citizen Developer** |
|---|---|---|---|
| **Sources reached** | All three | All three | Two — conversation denied |
| **Context retrieved** | Full set | Reduced | Materially reduced |
| **Confidential material** | Included | Withheld | Withheld |
| **Personal data** | Masked, count shown | Masked, count shown | Removed entirely |
| **Answer** | Complete, fully cited | Complete, flags one restricted source | Partial, states that material was withheld by policy |
| **Audit record** | Allowed | Allowed | Denied |

Three things make this persuasive rather than merely plausible:

- The reduction happens **before** synthesis — the restricted material never reaches the model, so
  the answer is not a filtered version of a fuller answer, it is a genuinely different answer built
  from genuinely different inputs.
- The system **says what it withheld and why**, rather than silently degrading.
- Every one of the three requests lands in the audit trail, with its access decision recorded.

Alongside this, the same view shows the authoritative policy document ranked above the casual
conversation that contradicts it — trust scoring visibly doing work.

### Demonstration B — the agent door

An external AI agent connects to ContextHub over the Model Context Protocol, is bound to a persona,
and asks the same question. Two things happen:

1. It receives the **same governed result** a human on that persona would receive.
2. The request appears **live in the audit trail**, marked as having arrived through the agent
   channel.

This is the demonstration that matters most strategically. It proves the governance path cannot be
bypassed by arriving through the machine interface — and it shows ContextHub operating as
infrastructure that other AI systems consume, which is the position we are claiming in the market.

### Supporting moments

- **The steward loop** — a human approves a proposed identity merge, and the graph changes in front
  of them. Human-in-the-loop curation, not an automated black box.
- **Provenance** — every statement in an answer resolves back to the specific source record it came
  from.

---

## 7. Phased approach

Six phases. Each has a single objective, and each is only complete when it has proved something
that the next phase depends on. The order is a genuine dependency chain, not a convenience.

### Phase 1 — Ground truth

**Objective.** Build the synthetic organization.

**What gets built.** A deterministic generator producing the three source corpora, with the planted
defects of Section 3 placed intentionally and recorded in a manifest.

**Proves.** That we have a defensible test bed — realistic enough to be credible, controlled enough
to be provable.

**Complete when.** Generation is reproducible; every planted defect is present and accounted for;
and the corpus verifiably contains no shortcut mapping between the different names for the same
thing.

### Phase 2 — Ingestion and resolution

**Objective.** Turn records into a connected graph.

**What gets built.** Three source connectors on a common interface; content segmentation and
indexing; entity extraction; the knowledge graph; and cross-source identity resolution with
confidence banding — high-confidence matches merge automatically, ambiguous ones queue for human
review, weak ones are discarded.

**Proves.** Claims 1 and 2 — unification without re-authoring, and automatic identity resolution.

**Complete when.** The same individual, arriving under three different names from three different
systems, resolves to one node; a code change links to the ticket it resolved; and genuinely
ambiguous matches are waiting for a human rather than silently guessed.

### Phase 3 — Governed retrieval

**Objective.** Build the core. This is the phase the POC exists for.

**What gets built.** Hybrid retrieval combining keyword search, semantic search and graph traversal;
result fusion and re-ranking; trust scoring; and then the governance layer — access control,
personal-data handling, hostile-content screening and audit logging.

Critically, all of this sits behind **one entry point**. Every consumer, human or machine, goes
through it. There is no second path.

**Proves.** Claim 3 — enforcement on the context itself.

**Complete when.** Demonstration A holds: the same question under three personas returns
progressively less context, the low-clearance persona receives nothing from the restricted source,
and three correctly-classified audit records exist. Injection attempts are blocked before reaching
any model.

### Phase 4 — Grounded answers

**Objective.** Turn governed context into a trustworthy answer.

**What gets built.** Answer synthesis constrained to only the retrieved material, with inline
citations that resolve to real source records, explicit acknowledgement when material was withheld,
and streaming delivery. An offline fallback produces a grounded answer without model access, so the
demonstration never depends on connectivity.

**Proves.** Claim 4 — grounded and traceable.

**Complete when.** Every citation in an answer resolves to a record that was actually retrieved —
no fabrication — and the offline path still produces something defensible.

### Phase 5 — Making it observable

**Objective.** Surface the working system through the existing interface.

**What gets built.** The prototype screens are connected to live data: real source status, live
querying, an explorable graph, the steward review queue, the audit trail, and adoption measures.
Every figure shown becomes a derived figure.

**Proves.** Nothing new — and that is the point. This phase makes the preceding four legible to a
non-technical audience. The capability exists after Phase 4; Phase 5 is what lets a room see it.

**Complete when.** The connected screens read entirely from the live system, and switching persona
visibly changes results.

### Phase 6 — The agent surface

**Objective.** Open the governed path to external AI agents.

**What gets built.** A Model Context Protocol server exposing retrieval and graph traversal as
tools, with persona bound at connection time — an agent inherits a clearance rather than choosing
one. Agent requests are tagged distinctly in the audit trail.

**Proves.** Claim 5 — and by extension, that the governance in Phase 3 is real rather than a
property of our own user interface.

**Complete when.** An external agent and a human on the same persona, asking the same question,
receive identical governed results, and both appear in the audit trail.

---

## 8. What approval means — acceptance criteria

Two criteria determine whether the POC succeeded. Everything else is supporting work.

**Criterion 1 — differential access.**
The same question, asked by three personas, returns three genuinely different bodies of context;
the lowest-clearance persona receives nothing from the restricted source; and all three access
decisions are correctly recorded in the audit trail.

**Criterion 2 — no side door.**
An external AI agent and a human user on the same persona receive *identical* governed results, and
both requests are audited. Any divergence between the two paths is treated as a failure, not a
discrepancy.

If both hold, the thesis is demonstrated and the direction is validated. If either fails, we have
learned something important before committing to a build.

---

## 9. What this POC deliberately does not prove

Stated plainly, so that the boundary is agreed up front rather than discovered later.

| Not proved | Why deferring is right |
|---|---|
| **Scale** | The corpus is thousands of records, not millions. Retrieval architecture at enterprise volume is a known engineering problem with known solutions — it is not where the risk lies |
| **Production connector engineering** | Authentication, rate limiting, incremental sync and deletion propagation are substantial work, but they are execution, not uncertainty |
| **Enterprise identity integration** | Persona is selected rather than authenticated. Binding to real directory groups and entitlements is the natural first increment after approval |
| **Retrieval quality at breadth** | Quality is tuned against a corpus we authored. Real-world quality requires real corpora and real users |
| **Multi-tenancy, cost model, availability** | All matter for a product. None of them tell us whether the core idea works |

The POC is scoped to answer one question: **does governed, persona-scoped, agent-accessible context
actually work end to end?** Everything above is deliberately excluded so that question gets a clean
answer.

---

## 10. Risks

| Risk | Mitigation |
|---|---|
| The synthetic organization reads as artificial under scrutiny | Write genuine prose in the policy documents and conversations rather than filler; review the corpus as content, not just as data |
| Retrieval quality too weak to make the persona contrast crisp | The corpus and the retrieval logic are developed together and tuned until the differential is unambiguous |
| The demonstration depends on network availability at the worst moment | Offline fallback built in from Phase 4, and rehearsed in that mode at least once |
| The agent path drifts from the human path over time | Enforced by an automated check — divergence breaks the build rather than being noticed later |
| Interface work expands and crowds out the core | Phase 5 is explicitly the most reducible phase. If schedule pressure appears, it narrows to the two screens that carry the demonstration |

---
