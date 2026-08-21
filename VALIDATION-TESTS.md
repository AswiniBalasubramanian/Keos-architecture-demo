# KEOS Architecture Builder — Validation Test Spec

Test cases for the "Build own Architecture" flow and its validation engine.

- **Target:** `KEOS-Architecture-standalone 1.html`
- **Run:** serve the file, **hard-reload the page**, paste the harness (§5) into the DevTools console, run `KEOS_TEST()`
- **Last run:** 24/24 passing

> **Run on a freshly loaded page.** Architectures and catalog descriptions live in memory for the session. Running the suite over a page you have already been clicking around in will leak state and produce false failures.

---

## 1. Severity model

Validation runs on every add / remove and produces two severities:

| Severity | Colour | Meaning | Blocks save? |
|---|---|---|---|
| **Risk** | Red, solid border | The architecture cannot work as drawn, or has a governance hole | Yes — Save becomes "Save anyway", needs a second press |
| **Advisory** | Amber, dashed border | Safe, but unverified, costly, or sprawling | No |

### Where a custom component lands

Adding a component that is **not in the KEOS catalog** is treated as *safe but unverified*:

- Chip renders with a **dashed amber** border and amber ⓘ
- Hover reads **"Unverified — …"**, not "Risk"
- Console lists it as **ADVISORY**
- It **does not** block Save
- Its fix action is **Review** (highlight), not **Fix with KEOS** (relocate), because there is no provably better home for it

A **known** component placed in the wrong layer stays a hard **Risk** — that is a real mistake, not an unknown.

---

## 2. What the validator checks

Findings come from three independent passes:

**A. Per-category** — one box at a time
- Critical category empty (Auth, Guardrails, Foundation Models, Vector Store, …)
- Multi-cloud mixing inside one category
- Redundancy — more than 5 overlapping tools

**B. Layer fit** — is this component in the right layer?
Each name resolves against the catalog to find its canonical category.
- Catalogued, wrong category → **Risk**
- Custom (user-added, uncatalogued) → **Advisory**
- Stock component missing from the catalog → **silent** (no baseline noise)

**C. Cross-layer** — does the stack hold together end to end?
23 rules across four chains:

| Chain | Example rule |
|---|---|
| **Execution** | Agents (06A) but no Foundation Model (05) → nothing to reason with |
| **Grounding** | Knowledge store (03) but no ingestion (02) → index never populated |
| **State** | Agents but no Agent Memory (04) → every interaction restarts cold |
| **Governance** | Agents but no Guardrails / Auth / Observability → unmitigated exposure |

---

## 3. Test cases

### Core validation

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-01 | Untouched AWS-based architecture | 0 risks (advisories allowed) | ✅ 0 risks |
| TC-02 | Empty **Foundation Models** | Cross-layer risk *and* critical-empty risk | ✅ 2 risks |
| TC-03 | Empty **Tool Protocol** | Risk: "can reason but cannot act" | ✅ |
| TC-04 | Empty **Eval & Simulation** | Risk: "regressions reach production unnoticed" | ✅ |
| TC-05 | Empty **Integration / ETL** + **Crawling** | Risk: ingestion path broken | ✅ |

### Layer fit and remediation

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-06 | Add **Neo4j** to *Invocation Surfaces* | Flagged **risk** — belongs in Graph Database | ✅ risk |
| TC-07 | Press **Fix with KEOS** on that finding | Neo4j relocates to *Graph Database* | ✅ moved |
| TC-11 | Stock component absent from catalog | Silent — no false positive | ✅ |

### Custom components → amber, never red

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-16 | Add custom "Acme Broker" with a description | `checkFit` returns level **warn** | ✅ warn |
| TC-17 | Same, then count risks | Risk count unchanged at 0 — Save not blocked | ✅ stayed 0 |
| TC-18 | Same, then read console | Appears as an **advisory** containing "unverified" | ✅ 3 advisories |
| TC-19 | Same | Advisory echoes the description you entered | ✅ echoed |
| TC-20 | One mismatch **and** one custom component | 1 risk + 1 custom advisory, independent | ✅ 1 / 1 |
| TC-21 | Fix action offered for a custom component | **Review** (`prune`), not move | ✅ prune |

### KEOS Context (Layer 04 — Memory) rules

Per `KEOS-Memory-Validation-Spec.md` §5.

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-22 | Empty **Compliance & PII** with memory/session present (A-MEM-05) | Advisory: interaction history stored with no PII control | ✅ |
| TC-23 | Empty **Data Quality Guards** with a feature store present (A-MEM-06) | Advisory: stale/drifted features undetected | ✅ |
| TC-24 | Same component name in both **Agent Memory** and **Session / KV Store** (R-MEM-02) | Risk: durable memory and disposable session state must not share one instance | ✅ |

### Advisories

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-08 | Azure Cosmos DB into an AWS *Session / KV Store* | Advisory: mixes AWS + Azure | ✅ |
| TC-09 | Push a category past 5 tools | Advisory: consolidation candidate | ✅ 9 tools |

### Metadata and suggestions

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-10 | `techMeta('Neo4j')` | Returns name, category, description | ✅ Graph Database |
| TC-13 | Suggestions for an AWS architecture | Same-cloud/neutral first, rival-cloud second | ✅ 16 rec / 1 other |
| TC-14 | Previously-sparse categories | ≥8 options each | ✅ 9, 12, 12 |

### Multi-architecture and state

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-12 | Build a 2nd architecture, edit it | 1st untouched — no overwrite | ✅ isolated |
| TC-15 | Save, edit further, Cancel | Reverts to last **saved** state, not original | ✅ |

---

## 4. Manual UI checks

Not covered by the harness — verify by hand:

| # | Step | Expected |
|---|---|---|
| M-1 | Hover any chip | Tooltip: name, category, description |
| M-2 | Hover a **red** chip | Tooltip adds **Risk —** in red |
| M-3 | Hover an **amber** chip | Tooltip adds **Unverified —** in amber, echoing your description |
| M-4 | Click a console row | Canvas scrolls to that group and flashes |
| M-5 | Type `ident` in Add → Custom name | Suggests Google Identity Platform, SAP Cloud Identity, Ping Identity with categories |
| M-6 | Pick a suggestion | Description field hides (already catalogued) |
| M-7 | Type a brand-new name | Description field appears and is **required** |
| M-8 | Submit new name with no description | Blocked, field turns red, toast prompts for it |
| M-9 | Save with only amber advisories | Saves immediately — no "Save anyway" |
| M-10 | Save with a red risk | Button becomes "Save anyway", needs a 2nd press |
| M-11 | Save cleanly | Exits to read-only view; name bar shows "Saved" + ✎ Edit |
| M-12 | Press "+ Build own Architecture" twice | Two separate pills, both independently editable |

---

## 5. Test harness

Hard-reload the page, paste into the DevTools console, then run `KEOS_TEST()`.
It creates a throwaway architecture, runs every case against a pristine snapshot, and cleans up after itself.

```js
window.KEOS_TEST = function(){
  const results = [];
  const t = (id, name, fn) => {
    let pass=false, detail='';
    try { const r = fn(); pass = r === true || (r && r.pass); detail = (r && r.detail) || ''; }
    catch(e){ pass=false; detail='ERROR: '+e.message; }
    results.push({id, name, pass, detail});
  };
  current='aws';
  const a = createArch('TEST-ARCH','aws');
  current='custom:'+a.id; editMode=true; render();
  const ORIG = snapshotOf(a);                 // pristine baseline, never mutated
  const P = l => pathForLabel(l);
  const tools = l => pathTarget(P(l)).map(x=>x.n);
  const empty = l => { pathTarget(P(l)).length = 0; };
  const reset = () => { restoreInto(a, ORIG); a.saved = snapshotOf(a); render(); };
  const risks = () => validateAll().filter(i=>i.level==='risk');
  const warns = () => validateAll().filter(i=>i.level==='warn');
  const msgs  = () => validateAll().map(i=>i.msg);
  const addCustom = (label, name, desc) => {
    const tt = T(name); tt.custom = true; if (desc) tt.desc = desc;
    pathTarget(P(label)).push(tt); return tt;
  };

  t('TC-01','Baseline has zero risks', ()=>{ reset(); const r=risks(); return {pass:r.length===0, detail:r.length+' risks'}; });
  t('TC-02','Empty Foundation Models', ()=>{ reset(); empty('Foundation Models'); const m=msgs().join('|');
    return {pass:/agents have nothing to reason with/.test(m) && /no model selected/.test(m), detail:risks().length+' risks'}; });
  t('TC-03','Empty Tool Protocol', ()=>{ reset(); empty('Tool Protocol');
    return {pass:/cannot act on enterprise systems/.test(msgs().join('|'))}; });
  t('TC-04','Empty Eval & Simulation', ()=>{ reset(); empty('Eval & Simulation');
    return {pass:/no evaluation harness/.test(msgs().join('|'))}; });
  t('TC-05','Empty ETL + Crawl', ()=>{ reset(); empty('Integration / ETL'); empty('Crawling & Extraction');
    return {pass:/no ingestion path/.test(msgs().join('|'))}; });
  t('TC-06','Known component in wrong layer = RISK', ()=>{ reset(); pathTarget(P('Invocation Surfaces')).push(T('Neo4j'));
    const f=checkFit('Neo4j','Invocation Surfaces');
    return {pass:!!f && f.level==='risk' && /Graph Database/.test(f.msg), detail:f?f.level:'none'}; });
  t('TC-07','Fix with KEOS moves component', ()=>{ reset(); pathTarget(P('Invocation Surfaces')).push(T('Neo4j'));
    const iss=validateAll().filter(i=>i.fix&&i.fix.kind==='move')[0];
    const from=P(iss.fix.label), to=P(iss.fix.target);
    const arr=pathTarget(from); const ix=arr.map(x=>x.n).indexOf(iss.fix.tool);
    const mv=arr.splice(ix,1)[0]; pathTarget(to).push(mv);
    return {pass: tools('Invocation Surfaces').indexOf('Neo4j')===-1 && tools('Graph Database').indexOf('Neo4j')!==-1,
            detail:'moved to '+iss.fix.target}; });
  t('TC-08','Multi-cloud mixing advisory', ()=>{ reset(); pathTarget(P('Session / KV Store')).push(T('Azure Cosmos DB'));
    return {pass:/mixes/.test(msgs().join('|'))}; });
  t('TC-09','Redundancy advisory', ()=>{ reset(); const p=P('Tool Protocol');
    ['A','B','C','D','E','F'].forEach(n=>pathTarget(p).push(T('Tool '+n)));
    return {pass:/overlapping tools/.test(msgs().join('|')), detail:pathTarget(p).length+' tools'}; });
  t('TC-10','techMeta metadata', ()=>{ const m=techMeta('Neo4j');
    return {pass:m.category==='Graph Database' && m.desc.length>10, detail:m.category}; });
  t('TC-11','Stock uncatalogued component stays silent', ()=>{ reset();
    return {pass: checkFit('Acme Widget','Invocation Surfaces')===null, detail:'no baseline noise'}; });
  t('TC-12','Architectures isolated', ()=>{ const b=createArch('TEST-ARCH-2','gcp');
    const prev=current; current='custom:'+b.id;
    pathTarget(P('Invocation Surfaces')).push(T('ZZZ-Marker'));
    const inB=tools('Invocation Surfaces').indexOf('ZZZ-Marker')!==-1;
    current=prev; const inA=tools('Invocation Surfaces').indexOf('ZZZ-Marker')!==-1;
    customArchs=customArchs.filter(x=>x.id!==b.id);
    return {pass: inB && !inA, detail:'B has marker, A does not'}; });
  t('TC-13','Base cloud grouping', ()=>{ reset(); const o=optionsForPath(P('Observability'));
    const rival=o.others.every(x=>{const c=cloudOf(x.n); return c && c!=='AWS';});
    return {pass:o.baseCloud==='AWS' && rival, detail:o.recommended.length+' rec / '+o.others.length+' other'}; });
  t('TC-14','Catalog depth', ()=>{ reset();
    const c=['Tool Protocol','Foundation Models','Registry & Lifecycle']
      .map(l=>{const o=optionsForPath(P(l)); return o.recommended.length+o.others.length;});
    return {pass:c.every(n=>n>=8), detail:c.join(',')}; });
  t('TC-15','Cancel reverts to saved', ()=>{ reset(); empty('Observability'); a.saved=snapshotOf(a);
    pathTarget(P('FinOps')).push(T('XYZ')); restoreInto(a,a.saved);
    return {pass: tools('FinOps').indexOf('XYZ')===-1 && tools('Observability').length===0, detail:'reverted'}; });

  // --- custom component = amber advisory, never a red risk ---
  t('TC-16','Custom component is WARN not RISK', ()=>{ reset();
    const tt=addCustom('Authentication & Authorization','Acme Broker','Brokers entitlement checks.');
    const f=checkFit(tt.n,'Authentication & Authorization',tt);
    return {pass: !!f && f.level==='warn', detail: f?f.level:'none'}; });
  t('TC-17','Custom component does not block Save', ()=>{ reset();
    const before=risks().length;
    addCustom('Authentication & Authorization','Acme Broker','Brokers entitlement checks.');
    return {pass: risks().length===before && before===0, detail:'risks stayed at '+before}; });
  t('TC-18','Custom component appears as advisory', ()=>{ reset();
    addCustom('Authentication & Authorization','Acme Broker','Brokers entitlement checks.');
    return {pass:/Acme Broker[\s\S]*unverified/.test(warns().map(i=>i.msg).join('|')), detail:warns().length+' advisories'}; });
  t('TC-19','Description carried into the advisory', ()=>{ reset();
    addCustom('Authentication & Authorization','Acme Broker','Brokers entitlement checks.');
    return {pass:/Recorded as: Brokers entitlement checks\./.test(warns().map(i=>i.msg).join('|')), detail:'desc echoed'}; });
  t('TC-20','Risk and warn coexist independently', ()=>{ reset();
    pathTarget(P('Invocation Surfaces')).push(T('Neo4j'));
    addCustom('Authentication & Authorization','Acme Broker','Brokers entitlement checks.');
    const r=risks().length, w=warns().filter(i=>/Acme Broker/.test(i.msg)).length;
    return {pass: r===1 && w===1, detail:r+' risk / '+w+' custom advisory'}; });
  t('TC-21','Custom advisory offers Review not Move', ()=>{ reset();
    addCustom('Authentication & Authorization','Acme Broker','Brokers entitlement checks.');
    const iss=validateAll().filter(i=>/Acme Broker/.test(i.msg))[0];
    return {pass: iss && iss.fix && iss.fix.kind==='prune', detail: iss?iss.fix.kind:'none'}; });

  t('TC-22','A-MEM-05: memory/session with no PII control', ()=>{ reset();
    pathTarget(P('Compliance & PII')).length = 0;
    return {pass:/no PII control in the architecture/.test(warns().map(i=>i.msg).join('|'))}; });
  t('TC-23','A-MEM-06: feature store with no data quality guard', ()=>{ reset();
    pathTarget(P('Data Quality Guards')).length = 0;
    return {pass:/feature\/context store is defined with no data quality guards/.test(warns().map(i=>i.msg).join('|'))}; });
  t('TC-24','R-MEM-02: same store backs memory and session state', ()=>{ reset();
    pathTarget(P('Session / KV Store')).push(T('Mem0'));
    return {pass:/backs both Agent Memory and Session \/ KV Store/.test(risks().map(i=>i.msg).join('|'))}; });

  customArchs=customArchs.filter(x=>x.id!==a.id);
  current='aws'; editMode=false; render();
  console.table(results);
  return results;
};
```

Summary line:

```js
(function(){ const r=KEOS_TEST(); console.log(r.filter(x=>x.pass).length+'/'+r.length+' passed'); })();
```

---

## 6. Known limitations

1. **Catalog-bound.** Layer-fit only *hard-fails* components in the catalog. Custom components are advisory-only — you are trusted, but warned. Their descriptions are recorded, not yet parsed to infer a category.
2. **Cloud detection is name-based.** `cloudOf()` matches vendor prefixes. A renamed component ("Our Bedrock Proxy") may be misattributed.
3. **Some catalog icons are unverified.** A few Iconify slugs were guessed and fall back to lettered monograms.
4. **No persistence.** Architectures live in memory; a reload clears them — which is also why the suite needs a fresh page.
