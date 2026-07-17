# Workflow templates — Codex blended in (Pattern A)

Copy-paste starting points. Each template is a complete Workflow script (begins
with `export const meta`). Adapt the prompts, schemas, and item lists to the task.

The **canonical `codexNode` helper is defined ONCE at the top of this file** — it
is the single source of truth. Each template below assumes it is in scope and
shows only a `/* paste the codexNode helper from the top of this file */` stub.
In a real script, define the helper once near the top.

## Contents

- **The `codexNode` helper** — the canonical relay; define once, the templates stub it.
- **Template 1 — Cross-model review** (find → Codex-verify): the default; fan-out find, per-finding adversarial verify.
- **Template 2 — Judge panel with a Codex juror**: candidate generation + mixed jury.
- **Template 3 — Cross-check one risky conclusion**: minimal; one cross-check of a single conclusion.
- **Template 4 — Loop-until-dry with a Codex gate**: iterate find → verify until a round adds no new confirmed findings.
- **Variants & notes**: batch node, large-payload (`revalidate:false`), `_codex_error` discipline.

---

## The `codexNode` helper

The single source of truth — `schema` drives both Codex's `--output-schema` and
(when `revalidate`) the Workflow's own re-validation, so the node returns a parsed
object.

```js
// One Workflow node, backed by a Codex headless run.
// schema:     a JS object (JSON Schema). Drives BOTH codex --output-schema and the
//             Workflow's re-validation (when revalidate), so this returns a parsed object.
// cwd:        optional working root (-C) so Codex can READ files itself (read-only still reads).
// effort:     optional reasoning-effort knob, mapped to -c model_reasoning_effort="..."
//             (low→xhigh; GPT-5.6 tiers also take "max", and "ultra" on sol/terra).
// model:      optional -m override; use fully-qualified tier IDs (gpt-5.6-sol/-terra/-luna).
// revalidate: default true; set false for large payloads and JSON.parse the string yourself.
//             Re-validation wraps the result in a { result: anyOf[schema, sentinel] }
//             envelope (unwrapped before return) so a legitimate {"_codex_error":true}
//             (e.g. after a watchdog kill) survives strict validation — keep filtering
//             the sentinel out downstream as always. The sentinel pins _codex_error
//             to the literal true (enum: [true]): a plain boolean would let a stray
//             {"_codex_error": false} — matching neither the user schema nor the
//             error convention — slip through validation and read as data downstream. The envelope exists because
//             Anthropic's tool input_schema rejects anyOf/oneOf/allOf at the TOP level
//             (400 before any agent runs); nested one level down it is legal. Root-
//             relative $refs inside the schema ("#", "#/$defs/…", "#/properties/…")
//             are rebased onto the embedded location ("#/properties/result/anyOf/0…"):
//             wrapping moves the document root, so unrebased they'd resolve against
//             the envelope — broken refs for $defs, and for recursive self-refs a
//             validator that rejects every valid payload. Only JSON-pointer fragments
//             ("#", "#/…") are rebased; named-anchor refs ("#name", paired with
//             $anchor) are left intact — anchors resolve by name within the schema
//             resource, so they survive relocation and rewriting would corrupt them.
//             And only in SCHEMA positions: values under const/enum/default/examples
//             are data (a "$ref" there is a literal, not a reference), and property
//             maps (properties/$defs/…) are walked as name→schema maps so a property
//             literally named "$ref" or "const" is not misread as a keyword.
//             $recursiveRef/$dynamicRef are deliberately left untouched: OpenAI's
//             structured-outputs schema subset rejects them, so codex --output-schema
//             fails such a schema upfront and it never reaches re-validation.
//             Codex itself still gets the ORIGINAL schema file (there the schema IS
//             the root). Rebasing stops at any subschema carrying a string $id —
//             root or nested (e.g. under $defs): it forms its own schema resource,
//             its "#…" refs already resolve against that $id wherever it sits, and
//             rewriting them would corrupt it.
// timeoutMs:  watchdog deadline for the codex run. Default 1200000 (20 min); "ultra"
//             nodes default to 1800000 (30 min); non-finite or non-positive values fall
//             back to the default. Runtime scales steeply with tier × effort (measured
//             on real tasks, codex 0.144.0: sol@max ~8 min, sol@xhigh ~14 min, sol@ultra
//             ~17 min — the defaults budget headroom, not the mean). The run is ALWAYS
//             launched as a BACKGROUND Bash call — the Bash tool's foreground cap is
//             10 min (default 2), and a timeout kill fabricates {"_codex_error":true}.
//             The in-snippet watchdog TERMs codex and its descendant tree at the
//             deadline and KILLs survivors 10 s later. Teardown is layered. Layer 1:
//             codex is launched under bash job control (set -m), giving it its own
//             process group — group kills (kill -- -PGID) reach every descendant
//             atomically, INCLUDING reparented orphans (pgid survives reparenting)
//             and workers codex backgrounds before exiting NORMALLY: the trap ends
//             with an unconditional group -9 (a no-op when the group is empty, so
//             the normal path stays delay-free; the pgid cannot be recycled while
//             any member lives). On the deadline path, group survivors get whatever
//             grace codex's own death took — tearing down a timed-out node
//             prioritizes not leaking over graceful shutdown. Layer 2, for pgid
//             escapees (a tool calling setsid):
//             descendants are also captured TRANSITIVELY (kids(): BFS over repeated
//             pgrep -P, depth-capped) to a temp file BEFORE the TERM — codex runs
//             tools through an intermediate `bash -lc`, so the real workload sits at
//             depth 2+: when that bash dies on TERM its children reparent and any
//             single-level pkill -P would never reach them. The parent's death also
//             unblocks the wrapper's wait, whose
//             EXIT trap reaps the watchdog mid-grace — so the trap ends by sweeping
//             the captured list itself. Before each final sweep, regrow() re-expands
//             the capture from still-alive captured PIDs — not just the (possibly
//             dead) codex root — so work a TERM-resistant survivor forked during the
//             grace window is swept too (best-effort: a tree that keeps forking can
//             always race a POSIX sweep). Every DELAYED -9 goes through sweep9(), which
//             re-checks each captured PID's parentage first (reparented orphan —
//             ppid 1 — or still inside the captured tree): a PID recycled during the
//             grace window is not blindly SIGKILLed. Residual caveats, both bounded
//             and deliberate: (a) under a child-subreaper, orphans reparent to the
//             subreaper rather than PID 1 and can escape the sweep — preferred over
//             -9'ing a recycled PID; (b) a descendant that BOTH escapes the pgid (its
//             own setsid) AND outlives a NORMAL codex exit is not hunted — Layer 2's
//             capture only runs while codex is alive or on the deadline path, and such
//             a process is an intentional detached daemon indistinguishable from one a
//             tool meant to leave running (a read-only verify node should not be
//             spawning these anyway). Timed-out runs still attempt it. The watchdog is
//             reaped together with its blocked sleep
//             child (pre-captured in the same kill) — TERMing the subshell alone
//             would orphan a deadline-length sleep per successful node. Cancellation
//             (INT/TERM) exits 130/143 through the EXIT trap: cleanup runs exactly
//             once (tree-capture+TERM→KILL when codex is alive, 2 s grace —
//             best-effort, the wrapper is being torn down), and the interrupted wait
//             no longer falls through to the final cat, which would disguise a
//             cancelled node as a normal empty result. The kill -0 guard keeps the
//             normal exit path delay-free. (POSIX sleep/kill/pgrep/ps plus bash job
//             control, because macOS ships neither GNU `timeout` nor `setsid` — set -m
//             provides the killable group without either.)
function codexNode(taskText, { schema, sandbox = 'read-only', model, cwd, effort, revalidate = true, phase, label, timeoutMs } = {}) {
  const flags = [
    model  ? `-m ${model}` : '',
    cwd    ? `-C "${cwd}"` : '',   // quote: cwd is an arbitrary path and may contain spaces
    effort ? `-c model_reasoning_effort="${effort}"` : '',
  ].filter(Boolean).join(' ')
  const schemaJson = JSON.stringify(schema, null, 2)
  // base64-embed schema+task: base64's alphabet has no shell metacharacters and no
  // line that could close a heredoc, so untrusted task text (e.g. reviewed repo
  // content flowing through a finding into taskText) cannot break out of the heredoc
  // and execute in the relay's shell — outside codex's read-only sandbox.
  const schemaB64 = Buffer.from(schemaJson, 'utf8').toString('base64')
  const taskB64 = Buffer.from(taskText, 'utf8').toString('base64')
  const deadlineMs = Number.isFinite(timeoutMs) && timeoutMs > 0 ? timeoutMs : (effort === 'ultra' ? 1800000 : 1200000)
  const deadlineSec = Math.ceil(deadlineMs / 1000)
  const sentinel = { type: 'object', additionalProperties: false, required: ['_codex_error'], properties: { _codex_error: { type: 'boolean', enum: [true] } } }
  const DATA_KEYS = ['const', 'enum', 'default', 'examples']   // keyword values that hold DATA, not schemas
  const MAP_KEYS = ['properties', 'patternProperties', '$defs', 'definitions', 'dependentSchemas',
    'dependencies']   // draft-07 dependencies: name→schema map too (its array-of-names form passes through as strings)
  const rebaseRefs = (node, isMap) => {
    if (Array.isArray(node)) return node.map(v => rebaseRefs(v))
    if (!node || typeof node !== 'object') return node
    if (!isMap && typeof node.$id === 'string') return node    // own schema resource — its "#…" refs resolve against that $id
    return Object.fromEntries(Object.entries(node).map(([k, v]) => {
      if (isMap) return [k, rebaseRefs(v)]                     // map keys are property NAMES; values are schemas
      if (DATA_KEYS.includes(k)) return [k, v]                 // a "$ref" inside const/enum is data, not a reference
      if (k === '$ref' && typeof v === 'string' && (v === '#' || v.startsWith('#/')))
        return [k, `#/properties/result/anyOf/0${v.slice(1)}`]
      return [k, rebaseRefs(v, MAP_KEYS.includes(k))]
    }))
  }
  const relaySchema = revalidate
    ? { type: 'object', additionalProperties: false, required: ['result'],
        properties: { result: { anyOf: [rebaseRefs(schema), sentinel] } } }
    : undefined
  return agent(
    `You are a RELAY, not a solver. Do NOT attempt the task yourself, do NOT
reason about it, do NOT use your own judgment. Your only job is to run Codex on
the task below and return Codex's JSON verbatim. If you answer it yourself the
whole point — a second, independent model — is defeated.

Codex runs for many minutes at higher reasoning efforts (real-task runs measured
~8–17 min — past the 10-minute foreground Bash cap, let alone the 2-minute
default, and a timeout kill fabricates a fake error). So launch the snippet
below as ONE Bash call with run_in_background: true — NEVER as a foreground
call — then wait for the background task to exit (you are re-invoked when it
does). The watchdog inside the snippet kills codex after ${deadlineSec}s if it
runs away (TERM, then KILL 10 s later). The snippet prints nothing except the
final cat, so the task's collected output IS the JSON — retrieve it and return it.

The snippet (schema and task are base64-embedded and decoded to temp files —
base64 is delimiter- and metacharacter-proof, so untrusted task text can't break
out of a heredoc into the relay's shell; the exec makes "prints nothing" literal —
bash job-death notices like "Terminated: 15" would otherwise leak into the
collected output around the final JSON):
  SCHEMA=$(mktemp); TASK=$(mktemp); OUT=$(mktemp); KIDSFILE=$(mktemp)
  exec 2>/dev/null
  printf %s '${schemaB64}' | base64 -d > "$SCHEMA"
  printf %s '${taskB64}' | base64 -d > "$TASK"
  set -m
  codex exec --skip-git-repo-check -s ${sandbox} ${flags} \\
    --output-schema "$SCHEMA" -o "$OUT" - < "$TASK" >/dev/null 2>&1 &
  CODEX_PID=$!
  set +m
  kids() {
    F="$1"; D=0
    while [ -n "$F" ] && [ "$D" -lt 32 ]; do
      F=$(pgrep -P "$(echo $F | tr ' ' ',')" | tr '\\n' ' ')
      [ -n "$F" ] && printf '%s\\n' $F
      D=$((D+1))
    done
  }
  sweep9() {
    while read -r P; do
      PP=$(ps -o ppid= -p "$P" 2>/dev/null | tr -d ' ')
      [ -z "$PP" ] && continue
      if [ "$PP" = 1 ] || [ "$PP" = "$CODEX_PID" ] || grep -qx "$PP" "$KIDSFILE"; then
        kill -9 "$P" 2>/dev/null
      fi
    done < "$KIDSFILE"
  }
  regrow() {
    SURV=$(while read -r P; do kill -0 "$P" 2>/dev/null && printf '%s ' "$P"; done < "$KIDSFILE")
    [ -n "$SURV" ] && kids "$SURV" >> "$KIDSFILE"
  }
  ( sleep ${deadlineSec}
    kids "$CODEX_PID" > "$KIDSFILE"
    kill -- -"$CODEX_PID" 2>/dev/null; kill "$CODEX_PID" $(cat "$KIDSFILE") 2>/dev/null
    sleep 10
    kids "$CODEX_PID" >> "$KIDSFILE"
    regrow
    kill -9 -- -"$CODEX_PID" 2>/dev/null; kill -9 "$CODEX_PID" 2>/dev/null; sweep9 ) >/dev/null 2>&1 &
  WATCHDOG_PID=$!
  cleanup() {
    if kill -0 "$CODEX_PID" 2>/dev/null; then
      kids "$CODEX_PID" > "$KIDSFILE"
      kill -- -"$CODEX_PID" 2>/dev/null; kill "$CODEX_PID" $(cat "$KIDSFILE") 2>/dev/null
      sleep 2
      kill -9 "$CODEX_PID" 2>/dev/null
    fi
    kill "$WATCHDOG_PID" $(pgrep -P "$WATCHDOG_PID") 2>/dev/null
    kill -9 -- -"$CODEX_PID" 2>/dev/null
    if [ -s "$KIDSFILE" ]; then regrow; sweep9; fi
  }
  trap cleanup EXIT
  trap 'exit 130' INT
  trap 'exit 143' TERM
  wait "$CODEX_PID"
  cat "$OUT"

Return EXACTLY the contents of "$OUT" — no prose, no markdown fences. If "$OUT"
is empty or codex errored, return the literal: {"_codex_error": true}
If you are given a structured-output tool whose schema has a "result" field,
pass that JSON (or the error literal) as the value of "result".`,
    { label: label ?? 'codex', phase, schema: relaySchema },
  ).then(r => (r && typeof r === 'object' && r.result !== undefined) ? r.result : r)
}
```

---

## Template 1 — Cross-model review (find → Codex-verify)

The canonical Pattern A. Claude finds broadly across dimensions; each finding is
adversarially verified by Codex as soon as it surfaces — pipeline, no barrier.
Survivors are findings a *different model family* could not refute.

```js
export const meta = {
  name: 'cross-model-review',
  description: 'Claude finds issues across dimensions; Codex adversarially verifies each',
  phases: [{ title: 'Find' }, { title: 'Verify' }],
}

function codexNode(taskText, opts) { /* paste the codexNode helper from the top of this file */ }

const FINDINGS = {
  type: 'object', additionalProperties: false, required: ['findings'],
  properties: { findings: { type: 'array', items: {
    type: 'object', additionalProperties: false, required: ['id', 'title', 'detail', 'file'],
    properties: { id: {type:'string'}, title: {type:'string'}, detail: {type:'string'}, file: {type:'string'} },
  } } },
}
const VERDICT = {
  type: 'object', additionalProperties: false, required: ['refuted', 'confidence', 'reasoning'],
  properties: { refuted: {type:'boolean'}, confidence: {type:'number'}, reasoning: {type:'string'} },
}

const DIMENSIONS = [
  { key: 'correctness', findPrompt: 'Review the changed files for correctness bugs. Return structured findings.' },
  { key: 'security',    findPrompt: 'Review the changed files for security issues. Return structured findings.' },
  // …add the dimensions the task needs
]

const results = await pipeline(
  DIMENSIONS,
  d => agent(d.findPrompt, { label: `find:${d.key}`, phase: 'Find', schema: FINDINGS }),
  (review, d) => parallel((review?.findings ?? []).map(f => () =>
    codexNode(
      `Adversarially verify this finding. Default to refuted=true if you cannot confirm it from the evidence.\nTITLE: ${f.title}\nFILE: ${f.file}\nDETAIL: ${f.detail}`,
      { schema: VERDICT, phase: 'Verify', label: `codex:${d.key}:${f.id}` },
    ).then(v => ({ ...f, dimension: d.key, verdict: v })))),
)

const confirmed = results.flat().filter(Boolean)
  .filter(f => f.verdict && !f.verdict._codex_error && f.verdict.refuted === false)
return { confirmed, total_findings: results.flat().filter(Boolean).length }
```

> **Verifying large/multi-file evidence?** Don't inline file contents — pass
> `cwd: '<abs target dir>'` to `codexNode` and name the files in the prompt; Codex
> reads them itself under read-only. See SKILL.md → "Variant: let Codex read the files itself".

---

## Template 2 — Judge panel with a Codex juror

Generate N candidates from different angles (optionally one by Codex for true
diversity), then score each with a mixed jury so no candidate is judged only by
its own author's model. Synthesize from the winner.

```js
export const meta = {
  name: 'mixed-judge-panel',
  description: 'Diverse candidate solutions scored by a Claude+Codex jury',
  phases: [{ title: 'Generate' }, { title: 'Judge' }, { title: 'Synthesize' }],
}

function codexNode(taskText, opts) { /* paste the codexNode helper from the top of this file */ }

const SOLUTION = {
  type: 'object', additionalProperties: false, required: ['approach', 'plan'],
  properties: { approach: {type:'string'}, plan: {type:'string'} },
}
const SCORE = {
  type: 'object', additionalProperties: false, required: ['score', 'rationale'],
  properties: { score: {type:'number', description:'0..10'}, rationale: {type:'string'} },
}

const TASK = 'Design an approach to <the problem>.'
const ANGLES = [
  { key: 'mvp',  prompt: `${TASK}\nBias toward the simplest shippable version.` },
  { key: 'risk', prompt: `${TASK}\nBias toward de-risking the hardest unknown first.` },
]

phase('Generate')
// Claude candidates + one Codex candidate for model diversity
const candidates = (await parallel([
  ...ANGLES.map(a => () => agent(a.prompt, { label: `gen:${a.key}`, schema: SOLUTION })
    .then(s => ({ ...s, author: `claude:${a.key}` }))),
  () => codexNode(`${TASK}\nPropose the approach you think is most robust.`,
        { schema: SOLUTION, label: 'gen:codex' }).then(s => ({ ...s, author: 'codex' })),
])).filter(Boolean).filter(c => !c._codex_error)   // an errored generator is not a candidate

phase('Judge')
const judged = await parallel(candidates.map((c, i) => () => parallel([
  () => agent(`Score this approach 0..10 for the problem.\n${JSON.stringify(c)}`, { label: `judge:claude:${i}`, schema: SCORE }),
  () => codexNode(`Score this approach 0..10 for the problem.\n${JSON.stringify(c)}`, { schema: SCORE, label: `judge:codex:${i}` }),
]).then(([claudeScore, codexScore]) => {
  // tag each juror with its model family and drop errored ones (never score an infra
  // failure as 0). The panel's whole point is that no candidate is judged only by its
  // OWN family: if the cross-family juror errored and only the author's family survived,
  // that is not a valid verdict — avg null (excluded from ranking), same as all-errored.
  const authorFamily = c.author.split(':')[0]
  const valid = [
    claudeScore && !claudeScore._codex_error ? { ...claudeScore, family: 'claude' } : null,
    codexScore  && !codexScore._codex_error  ? { ...codexScore,  family: 'codex'  } : null,
  ].filter(Boolean)
  const jury = valid.some(s => s.family !== authorFamily) ? valid : []
  const avg = jury.length ? jury.reduce((a, s) => a + (s.score ?? 0), 0) / jury.length : null
  return { candidate: c, avg, scores: jury }
})))

const ranked = judged.filter(Boolean).filter(j => j.avg !== null).sort((a, b) => b.avg - a.avg)
const winner = ranked[0]
if (!winner) { log('no candidate got a valid cross-family verdict — nothing rankable'); return { final: null, winner: null, ranking: [] } }

phase('Synthesize')
const final = await agent(
  // pass the runners-up's FULL content, not just author+avg — "graft the best ideas"
  // is impossible if the synthesizer can only see who ranked where, not what they proposed.
  `Write the final approach. Base it on the winner, grafting the best ideas from the runners-up.\n` +
  `WINNER: ${JSON.stringify(winner.candidate)}\n` +
  `RUNNERS_UP: ${JSON.stringify(ranked.slice(1).map(j => j.candidate))}`,
)
return { final, winner: winner.candidate.author, ranking: ranked.map(j => ({ author: j.candidate.author, avg: j.avg })) }
```

---

## Template 3 — Cross-check one risky conclusion

Minimal blend: Claude does the work, and a single Codex node tries to refute the
headline conclusion before reporting it. Cheap insurance for high-stakes claims.

```js
export const meta = {
  name: 'codex-crosscheck',
  description: 'Claude concludes; Codex attempts to refute before reporting',
  phases: [{ title: 'Conclude' }, { title: 'Cross-check' }],
}

function codexNode(taskText, opts) { /* paste the codexNode helper from the top of this file */ }

const VERDICT = {
  type: 'object', additionalProperties: false, required: ['refuted', 'confidence', 'reasoning'],
  properties: { refuted: {type:'boolean'}, confidence: {type:'number'}, reasoning: {type:'string'} },
}

phase('Conclude')
const conclusion = await agent('Analyze <X> and state your single headline conclusion with evidence.')

phase('Cross-check')
const verdict = await codexNode(
  `A Claude analysis concluded the following. Adversarially try to refute it; default to refuted=true if the evidence is not airtight.\n\n${conclusion}`,
  { schema: VERDICT, label: 'codex:crosscheck' },
)
return { conclusion, codex_verdict: verdict, trustworthy: verdict && !verdict._codex_error && verdict.refuted === false }
```

---

## Template 4 — Loop-until-dry with a Codex gate

Iterate Claude-find → Codex-verify until a round adds no new *confirmed* finding
(or a max-rounds cap), feeding already-known ids back into the finder as "don't
repeat". Exhaustion is judged on *confirmed* survivors, not raw Claude output — a
round whose findings Codex refutes wholesale is just as dry as a round with none.

```js
export const meta = {
  name: 'loop-until-dry-codex',
  description: 'Iterate Claude-find -> Codex-verify until a round adds no new confirmed findings',
  phases: [{ title: 'Hunt' }],
}

function codexNode(taskText, opts) { /* paste the codexNode helper from the top of this file */ }

const FINDINGS = {
  type: 'object', additionalProperties: false, required: ['findings'],
  properties: { findings: { type: 'array', items: {
    type: 'object', additionalProperties: false, required: ['id', 'title', 'detail'],
    // the finder assigns a STABLE id so rounds can dedupe.
    properties: { id: {type:'string'}, title: {type:'string'}, detail: {type:'string'} },
  } } },
}
const VERDICT = {
  type: 'object', additionalProperties: false, required: ['refuted', 'reasoning'],
  properties: { refuted: {type:'boolean'}, reasoning: {type:'string'} },
}

const MAX_ROUNDS = 4          // Workflow JS forbids unbounded loops / Date.now / Math.random — always cap.
const confirmed = []
const seen = new Set()        // stable ids already surfaced
let roundsUsed = 0

for (let round = 0; round < MAX_ROUNDS; round++) {
  roundsUsed = round + 1
  phase(`Round ${roundsUsed}`)
  const known = [...seen].join(', ') || '(none yet)'
  const review = await agent(
    `Find issues in <the target>. Assign each a STABLE id. Do NOT repeat already-known ids: ${known}. Return structured findings.`,
    { label: `find:r${round}`, schema: FINDINGS },
  )
  const fresh = (review?.findings ?? []).filter(f => !seen.has(f.id))
  fresh.forEach(f => seen.add(f.id))
  const verdicts = fresh.length ? await parallel(fresh.map(f => () =>
    codexNode(`Adversarially verify, defaulting to refuted=true if uncertain:\n${f.title}\n${f.detail}`,
      { schema: VERDICT, label: `codex:${f.id}` }).then(v => ({ ...f, verdict: v })))) : []
  const usable = verdicts.filter(Boolean).filter(f => f.verdict && !f.verdict._codex_error)
  // an errored verdict is NO verdict: a round whose verdicts ALL errored is an infra
  // failure (auth expiry, bad schema, watchdog kill) — surface it, never exit as "dry".
  if (fresh.length > 0 && usable.length === 0)
    throw new Error(`round ${roundsUsed}: every Codex verdict errored — re-run the preflight`)
  // partially errored/lost verdicts: un-see those findings so a later round can
  // re-surface them — a transient failure must not permanently drop an unverified finding.
  const usableIds = new Set(usable.map(f => f.id))
  const unjudged = fresh.filter(f => !usableIds.has(f.id))
  unjudged.forEach(f => seen.delete(f.id))
  if (unjudged.length)
    log(`round ${roundsUsed}: ${unjudged.length}/${fresh.length} verdicts errored or lost — returned to the pool`)
  const newlyConfirmed = usable.filter(f => f.verdict.refuted === false)
  confirmed.push(...newlyConfirmed)
  // the gate is CONFIRMED survivors, not raw Claude output — and only a FULLY-judged
  // round may declare dry: every fresh finding has a usable verdict and none was
  // confirmed. (A round with no fresh findings is trivially dry.)
  if (newlyConfirmed.length === 0 && unjudged.length === 0) break
}
return { confirmed, rounds_used: roundsUsed }
```

---

## Variants & notes

### Batch codex node — verify/judge N small items in one Codex run

Every template above fans out one Codex node per item. When items are **small and
homogeneous** and a single Codex context can hold them, batch them instead: one
slot, one Codex/OpenAI run, results keyed back by id. Fan out (Template 1) only
when each item needs a fresh, independent context — or when items are large.

```js
// instruction: what to do per item. items: [{ id, ... }]. schema: the BATCH shape below.
function codexBatchNode(instruction, items, opts = {}) {
  const taskText = `${instruction}
Return EXACTLY one result object per input id, in a "results" array — same ids, no extras.
INPUT ITEMS (JSON array):
${JSON.stringify(items)}`
  return codexNode(taskText, opts)  // returns the parsed { results: [...] }
}

const BATCH_VERDICTS = {
  type: 'object', additionalProperties: false, required: ['results'],
  properties: { results: { type: 'array', items: {
    type: 'object', additionalProperties: false, required: ['id', 'refuted', 'reasoning'],
    properties: { id: {type:'string'}, refuted: {type:'boolean'}, reasoning: {type:'string'} },
  } } },
}

// usage: verify all findings in ONE codex run, then key verdicts back by id.
const { results = [] } = await codexBatchNode(
  'Adversarially verify each finding; default refuted=true if uncertain.',
  findings.map(f => ({ id: f.id, title: f.title, detail: f.detail })),
  { schema: BATCH_VERDICTS, label: 'codex:batch-verify' },
) || {}
const byId = new Map(results.map(r => [r.id, r]))
```

### Large payloads — skip Workflow re-validation, parse the string yourself

`revalidate: true` (default) re-validates Codex's JSON through the Claude wrapper
and auto-parses it — best for small verdicts/scores, immune to schema drift. For a
**large** structured output, set `revalidate: false` so the wrapper relays the raw
string (Codex's own `--output-schema` still enforces shape), then parse it
yourself with the same `_codex_error` convention:

```js
const raw = await codexNode(bigTask, { schema: BIG_SCHEMA, revalidate: false, label: 'codex:big' })
let obj
try { obj = JSON.parse(raw) } catch { obj = { _codex_error: true } }
```

### `_codex_error` discipline (applies to every template)

A Codex node returns `{ _codex_error: true }` on failure. Treat it as **"no data"**,
never as a pass/refute:

- Always `.filter(Boolean)` to drop null agent returns, **then** drop any result
  where `_codex_error` is true, **before** any aggregation.
- In a judge panel, **exclude** errored jurors from the average — scoring them `0`
  silently penalizes the candidate for an infra failure. An **all-errored jury is
  no verdict at all**: mark it (`avg: null`) and keep the candidate out of the
  ranking rather than letting an implicit `0` sort it last (Template 2 does both).
- If more than a small fraction of nodes return `_codex_error` in a run, stop and
  re-run the preflight — that signals auth expiry or a bad schema, not flakiness.

Full failure-mode table in `codex-headless.md` → Troubleshooting.

### Notes that apply to all templates

- **Keep Codex nodes short** (read-only verify/judge/small-gen). They hold a
  concurrency slot for Codex's full runtime — do not put long implementations here.
- **Give codex real time.** The helper launches every codex run as a background
  Bash call with a sleep+kill watchdog: `timeoutMs` defaults to 20 min, and
  `ultra` nodes get 30 (measured real-task runs: sol@`max` ~8 min, sol@`xhigh`
  ~14 min, sol@`ultra` ~17 min — the defaults budget headroom, not the mean).
  A foreground Bash call — 10-minute cap, 2-minute default — would kill
  high-effort runs mid-flight and fabricate `{"_codex_error":true}`.
- **Schemas must be strict** — set `additionalProperties: false` **and** list
  *every* key from `properties` in `required` (OpenAI's structured-output backend
  400s on a partial `required` *before the run starts* — strict mode has no
  optional keys). Strictness also makes both Codex's `--output-schema` and the
  Workflow re-validation reliable.
- **Tier & effort knobs — pick per node, the choice is open.** `effort:` maps to
  `-c model_reasoning_effort` (`low`→`xhigh`, plus `max` on all GPT-5.6 tiers
  and `ultra` on sol/terra); `model:` maps to `-m` (fully-qualified tier IDs:
  `gpt-5.6-sol` flagship / `gpt-5.6-terra` balanced / `gpt-5.6-luna` fast-cheap).
  Wide verify fan-outs like Template 1 pair well with `model: 'gpt-5.6-luna'` +
  `effort: 'low'`; a load-bearing single verdict (Template 3) with
  `gpt-5.6-sol` at `high`/`xhigh` or `max` — a structured verify is a few K
  tokens, so the high end is a real option even at flagship rates. Reserve
  `ultra` (auto sub-agent delegation, long runs) for one decisive node, never a
  fan-out. Tier table and caveats: `codex-headless.md` →
  "Model tiers & reasoning effort".
