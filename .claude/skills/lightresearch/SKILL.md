---
name: lightresearch
description: "Быстрый поиск фактов с источниками: 2-3 угла поиска, топ-5 источников, однопроходная проверка каждого утверждения. ~15-20 агентов. Для внутренней сверки, не для клиентских публикаций."
user-invocable: true
---

# /lightresearch — быстрый сорсинг фактов

## Зачем

«Дай цифру с источником» не требует стоагентного глубокого ресёрча. Этот скилл делает тот же паттерн (scope → search → fetch → verify → synthesize), но с малым фан-аутом и однопроходной проверкой фактов вместо многоголосой верификации. Дёшево и быстро.

## Когда подходит, когда нет

**Подходит:**
- Результат для внутреннего использования — сверить, прикинуть, проверить себя
- Достаточно 1 разумного источника с датой
- Ошибка исправима — это не финальная версия для клиента

**НЕ подходит (эскалируй Сержу — у него есть тяжёлая версия с адверсариал-верификацией):**
- Результат идёт в клиентский отчёт или публикацию
- Цифру будут оспаривать — нужна многоголосая проверка
- Найдены противоречащие источники — light-прогон это покажет, дальше нужна глубина

## Как запустить

Вызови Workflow-инструмент с инлайн-скриптом ниже, `args` = вопрос (строка).

```js
export const meta = {
  name: 'lightresearch',
  description: 'Lightweight sourced lookup — few angles, single-pass fact check, no adversarial multi-vote.',
  phases: [
    { title: 'Scope', detail: 'Decompose question into 2-3 search angles' },
    { title: 'Search', detail: '2-3 parallel WebSearch agents' },
    { title: 'Fetch', detail: 'Fetch top 5 sources, extract claims' },
    { title: 'Verify', detail: 'Single-pass skeptical check per claim' },
    { title: 'Synthesize', detail: 'Merge dupes, rank confidence, cite sources' },
  ],
}

const VOTES_PER_CLAIM = 1
const REFUTATIONS_REQUIRED = 1
const MAX_FETCH = 5
const MAX_VERIFY_CLAIMS = 8

const SCOPE_SCHEMA = {
  type: "object", required: ["question", "angles", "summary"],
  properties: {
    question: { type: "string" },
    summary: { type: "string" },
    angles: { type: "array", minItems: 2, maxItems: 3, items: {
      type: "object", required: ["label", "query"],
      properties: { label: { type: "string" }, query: { type: "string" }, rationale: { type: "string" } },
    }},
  },
}
const SEARCH_SCHEMA = {
  type: "object", required: ["results"],
  properties: {
    results: { type: "array", maxItems: 4, items: {
      type: "object", required: ["url", "title", "relevance"],
      properties: { url: { type: "string" }, title: { type: "string" }, snippet: { type: "string" }, relevance: { enum: ["high", "medium", "low"] } },
    }},
  },
}
const EXTRACT_SCHEMA = {
  type: "object", required: ["claims", "sourceQuality"],
  properties: {
    sourceQuality: { enum: ["primary", "secondary", "blog", "forum", "unreliable"] },
    publishDate: { type: "string" },
    claims: { type: "array", maxItems: 4, items: {
      type: "object", required: ["claim", "quote", "importance"],
      properties: { claim: { type: "string" }, quote: { type: "string" }, importance: { enum: ["central", "supporting", "tangential"] } },
    }},
  },
}
const VERDICT_SCHEMA = {
  type: "object", required: ["refuted", "evidence", "confidence"],
  properties: { refuted: { type: "boolean" }, evidence: { type: "string" }, confidence: { enum: ["high", "medium", "low"] }, counterSource: { type: "string" } },
}
const REPORT_SCHEMA = {
  type: "object", required: ["summary", "findings", "caveats"],
  properties: {
    summary: { type: "string" },
    findings: { type: "array", items: {
      type: "object", required: ["claim", "confidence", "sources", "evidence"],
      properties: { claim: { type: "string" }, confidence: { enum: ["high", "medium", "low"] }, sources: { type: "array", items: { type: "string" } }, evidence: { type: "string" } },
    }},
    caveats: { type: "string" },
  },
}

phase("Scope")
const QUESTION = (typeof args === "string" && args.trim()) || ""
if (!QUESTION) return { error: "No research question provided." }
const scope = await agent(
  "Decompose this research question into 2-3 complementary search angles — this is a LIGHT lookup, not exhaustive research. Pick the 2-3 angles that will most directly answer it.\n\n## Question\n" + QUESTION + "\n\nStructured output only.",
  { label: "scope", schema: SCOPE_SCHEMA }
)
if (!scope) return { error: "Scope agent returned no result." }
log("Q: " + QUESTION.slice(0, 80))
log("Light scope: " + scope.angles.length + " angles — " + scope.angles.map(a => a.label).join(", "))

const normURL = u => { try { const p = new URL(u); return (p.hostname.replace(/^www\./, "") + p.pathname.replace(/\/$/, "")).toLowerCase() } catch { return u.toLowerCase() } }
const seen = new Map()
const relRank = { high: 0, medium: 1, low: 2 }
let fetchSlots = MAX_FETCH

const SEARCH_PROMPT = (angle) =>
  "## Web Searcher: " + angle.label + "\n\nQuestion: \"" + QUESTION + "\"\nAngle: " + angle.label + " — " + (angle.rationale || "") + "\nQuery: `" + angle.query + "`\n\nUse WebSearch. Return top 3-4 most relevant results, skip SEO spam. Structured output only."

const FETCH_PROMPT = (source, angle) =>
  "## Source Extractor\n\nQuestion: \"" + QUESTION + "\"\nURL: " + source.url + "\nTitle: " + source.title + "\nFound via: " + angle + "\n\nWebFetch the page. Assess source quality. Extract 1-4 falsifiable claims bearing on the question, each with a supporting quote and importance rating. If fetch fails/irrelevant, return claims: [] and sourceQuality: \"unreliable\". Structured output only."

const VERIFY_PROMPT = (claim) =>
  "## Quick fact-check (single pass, not adversarial)\n\nQuestion: " + QUESTION + "\nClaim: \"" + claim.claim + "\"\nSource: " + claim.sourceUrl + " (" + claim.sourceQuality + ")\nQuote: \"" + claim.quote + "\"\n\nCheck: (1) does the quote actually support the claim? (2) quick WebSearch — any obvious, specific contradiction? (3) is it clearly outdated for a fast-moving topic?\n\nThis is a LIGHT check — default to refuted=false unless you find a clear misread or a specific contradicting source. Uncertainty alone is not grounds to refute; rate it low confidence instead. Structured output only."

const searchResults = await pipeline(
  scope.angles,
  angle => agent(SEARCH_PROMPT(angle), { label: "search:" + angle.label, phase: "Search", schema: SEARCH_SCHEMA })
    .then(r => r ? { angle: angle.label, results: r.results } : null),
  searchResult => {
    if (!searchResult) return []
    const novel = [...searchResult.results].sort((a, b) => relRank[a.relevance] - relRank[b.relevance]).filter(r => {
      const key = normURL(r.url)
      if (seen.has(key) || fetchSlots <= 0) return false
      seen.set(key, true); fetchSlots--; return true
    })
    return parallel(novel.map(source => () => {
      let host = "unknown"; try { host = new URL(source.url).hostname.replace(/^www\./, "") } catch {}
      return agent(FETCH_PROMPT(source, searchResult.angle), { label: "fetch:" + host, phase: "Fetch", schema: EXTRACT_SCHEMA })
        .then(ext => ext ? { url: source.url, title: source.title, sourceQuality: ext.sourceQuality, publishDate: ext.publishDate, claims: ext.claims.map(c => ({ ...c, sourceUrl: source.url, sourceQuality: ext.sourceQuality })) } : null)
        .catch(() => ({ url: source.url, title: source.title, sourceQuality: "unreliable", claims: [] }))
    }))
  }
)

const allSources = searchResults.flat().filter(Boolean)
const allClaims = allSources.flatMap(s => s.claims)
const impRank = { central: 0, supporting: 1, tangential: 2 }
const qualRank = { primary: 0, secondary: 1, blog: 2, forum: 3, unreliable: 4 }
const rankedClaims = [...allClaims].sort((a, b) => (impRank[a.importance] - impRank[b.importance]) || (qualRank[a.sourceQuality] - qualRank[b.sourceQuality])).slice(0, MAX_VERIFY_CLAIMS)
log("Fetched " + allSources.length + " sources → " + allClaims.length + " claims → checking top " + rankedClaims.length)
if (allClaims.length > MAX_VERIFY_CLAIMS) log((allClaims.length - MAX_VERIFY_CLAIMS) + " lower-priority claims dropped by cap — a deeper research pass would be needed for full coverage")

if (rankedClaims.length === 0) {
  return { question: QUESTION, summary: "No claims extracted from " + allSources.length + " sources.", findings: [], sources: allSources.map(s => ({ url: s.url, quality: s.sourceQuality })) }
}

phase("Verify")
const voted = (await parallel(rankedClaims.map(claim => () =>
  agent(VERIFY_PROMPT(claim), { label: "check:" + claim.claim.slice(0, 40), phase: "Verify", schema: VERDICT_SCHEMA }).then(v => {
    if (!v) return { ...claim, verdict: null, survives: false, isRefuted: false }
    const mark = v.refuted ? "✗" : "✓"
    log("\"" + claim.claim.slice(0, 50) + "…\" " + mark + " (" + v.confidence + ")")
    return { ...claim, verdict: v, survives: !v.refuted, isRefuted: v.refuted }
  })
))).filter(Boolean)

const confirmed = voted.filter(c => c.survives)
const killed = voted.filter(c => c.isRefuted)
log("Check done: " + confirmed.length + " kept, " + killed.length + " dropped")

if (confirmed.length === 0) {
  return { question: QUESTION, summary: "All " + killed.length + " claims failed the fact-check. A deeper, adversarial research pass is recommended.", findings: [], refuted: killed.map(c => ({ claim: c.claim, evidence: c.verdict?.evidence })), sources: allSources.map(s => ({ url: s.url, quality: s.sourceQuality })) }
}

phase("Synthesize")
const block = confirmed.map((c, i) => "### [" + i + "] " + c.claim + "\nSource: " + c.sourceUrl + " (" + c.sourceQuality + ", confidence " + c.verdict.confidence + ")\nQuote: \"" + c.quote + "\"\nCheck: " + c.verdict.evidence + "\n").join("\n")

const report = await agent(
  "## Light synthesis\n\nQuestion: " + QUESTION + "\n\n" + confirmed.length + " claims passed a single-pass fact-check (not adversarial — treat 'high confidence' findings as solid, 'low confidence' as worth double-checking before quoting externally).\n\n" + block +
  "\n\n1. Merge duplicate claims, group into findings answering the question.\n2. Confidence per finding: high (primary source, clean check), medium (secondary source or minor caveat), low (blog/forum or thin evidence).\n3. 2-4 sentence summary.\n4. Caveats: note this was a light single-pass check, not adversarially verified — flag anything that should be re-verified before being quoted externally.\n\nStructured output only.",
  { label: "synthesize", schema: REPORT_SCHEMA }
)
if (!report) return { question: QUESTION, summary: "Synthesis skipped — returning " + confirmed.length + " raw checked claims.", confirmed: confirmed.map(c => ({ claim: c.claim, source: c.sourceUrl, quote: c.quote })), sources: allSources.map(s => ({ url: s.url, quality: s.sourceQuality })) }

return {
  question: QUESTION, ...report,
  refuted: killed.map(c => ({ claim: c.claim, evidence: c.verdict?.evidence })),
  sources: allSources.map(s => ({ url: s.url, quality: s.sourceQuality, claimCount: s.claims.length })),
  stats: { mode: "light", angles: scope.angles.length, sourcesFetched: allSources.length, claimsExtracted: allClaims.length, confirmed: confirmed.length, killed: killed.length, agentCalls: 1 + scope.angles.length + allSources.length + rankedClaims.length + 1 },
}
```

## Пример вызова

```
/lightresearch Средняя зарплатная вилка менеджера по продажам B2B в Москве 2026, с источником
```

Ожидаемый бюджет: ~1 (scope) + 2-3 (search) + ≤5 (fetch) + ≤8 (verify) + 1 (synth) ≈ **15-20 агентов**.

## Правило

В отчёте каждый факт помечен confidence. Всё, что `low`, перед использованием во внешнем материале — перепроверь сама или отдай Сержу на глубокую верификацию.
