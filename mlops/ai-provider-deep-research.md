# ai-provider-deep-research

## 📖 Description

Comprehensive multi-source deep research methodology for AI providers, models, and APIs — cross-validation, structured report compilation, cost/benchmark analysis, and Obsidian knowledge base integration.

---

# AI Provider Deep Research Methodology

## Core Principle: 深度研究 實用性 (Deep Verifiable Research x Practical Utility)

This user demands **real verified data** from real sources, not surface observations or generic summaries. Every piece of analysis must answer: **"Is this actually true, and does it matter for production?"**

**Do NOT:**
- Speculate without labeling it as speculation
- Present provider marketing claims as verified facts
- Give generic advice that could apply to any provider
- Accept benchmark numbers from a single source at face value
- Stop after extracting one page — follow citations, search for disagreements

**DO:**
- Cross-validate ALL claims across 3+ independent sources
- Flag discrepancies between official marketing and independent benchmarks
- Verify pricing, specs, and availability against actual API docs
- Test claims against multiple evaluation platforms (Claw-Eval, Artificial Analysis, SWE-bench, etc.)
- Give a clear, honest verdict for the user's specific use case

---

## Phase 1: Multi-Angle Initial Sweep

Begin with **3+ independent web_search queries** to maximize coverage. Each query should use a different angle:

```python
# Pattern — launch all three concurrently:
web_search(query="[Provider] blog articles site:[domain]")
web_search(query="[Provider] [product] documentation")
web_search(query='"[Provider]" "blog" OR "articles" OR "release" 2025 2026')
```

**Search angles to cover:**
1. **Official presence**: `site:provider.com blog`, docs pages, product pages
2. **Independent coverage**: news articles, case studies, third-party reviews
3. **Technical depth**: architecture blog posts, benchmark analyses, GitHub discussions
4. **Community reception**: Reddit, Discord, social media sentiment
5. **Financial/strategic**: fundraising, market positioning, competitive analysis
6. **Developer feedback**: API documentation, code examples, GitHub issues

---

## Phase 2: Deep Extraction + Critical Reading

Extract ALL key pages found in Phase 1, not just the top result:

```python
web_extract(urls=["https://provider.com/", "https://provider.com/docs", 
                   "https://blog.provider.com/key-article", ...])
```

**Critical reading checklist per source:**
- ✅ Is this first-party or third-party?
- ✅ Are benchmark numbers from the provider's own blog or an independent evaluator?
- ✅ Does the article contain disclaimers like "speculative", "projected", or "not verified"?
- ✅ What's the publication date — is it still current?
- ✅ Are there "disclaimer" notes that the content is sponsored or AI-generated?

**Flag these patterns:**
- 🟡 **Speculative articles** (e.g., "projected V4 features") — label clearly as speculation
- 🔴 **Provider-only benchmarks** — cross-check against independent sources
- 🟠 **Outdated information** — check if pricing, specs, or APIs have changed

---

## Phase 3: Iterative Depth — Follow the Trail

After Phase 2, you'll have uncovered references to **secondary sources**. Follow them:

```python
# Search for specific claims from Phase 2
web_search(query="[Provider] [specific model] benchmark [benchmark name]")
web_search(query="[Provider] [claim] independent verification")
web_search(query="[Provider] vs [competitor] comparison")

# Search for missing pieces
web_search(query="[Provider] pricing API 2026")
web_search(query="[Provider] architecture MoE routing")
```

**When to stop:** when the same facts appear repeatedly across independent sources AND you've confirmed the key outliers (discrepancies between official and independent data).

---

## Phase 4: Cross-Validation Matrix

For every material claim, track which sources agree/disagree:

| Claim | Official Source | Independent Source 1 | Independent Source 2 | Verdict |
|-------|----------------|---------------------|---------------------|---------|
| Price $X/M tokens | ✅ API docs | ✅ third-party | ✅ pricing page | ✅ Confirmed |
| Benchmark Y% | 🔴 60.9% | 🟠 51.8% | — | ⚠️ Discrepancy — investigate |
| Context window | ✅ 256K | ✅ 256K | ✅ docs | ✅ Confirmed |

**Common discrepancies to watch for:**
- **Benchmark version differences**: Pass³ vs Pass@3, different test sets, different evaluator harnesses (SWE-bench Verified vs DeepSWE can differ by 70+ pts)
- **Pricing changes**: Free promotional periods, "permanent" discounts that may change
- **Spec changes**: Context window reductions (1M → 256K), model deprecation dates
- **Capability claims**: "Supports X" vs "works reliably on X"

---

## Phase 5: Structured Report Compilation

Organize findings into a consistent structure:

```markdown
# [Provider] 全面深度研究報告

## 1. 公司概覽 (table: HQ, founded, team size, funding, mission)

## 2. 創辦人/團隊 (if founder-driven company)

## 3. 募資歷程 (timeline table)

## 4. 產品生態系 (key products, what they replace)

## 5. 模型家族 (model table: specs, context, pricing, capabilities)

## 6. 基準測試表現 (benchmark tables, include competitor comparisons)

## 7. 定價策略 (pricing table, cost comparison vs competitors)

## 8. 技術架構深潛 (architecture innovations, routing, training methods)

## 9. 基礎設施與合作夥伴 (cloud providers, integrations)

## 10. 市場策略與定位 (target users, differentiators, geopolitical angle)

## 11. 成長指標 (timeline of users, revenue, DAU)

## 12. 風險與注意事項 (numbered list, honest assessment)

## 13. [Your Role] 戰略意義 (actionable recommendations)

## 14. 外部參考連結
```

**Required sections for every report:**
- **Benchmark section** with competitor comparisons, not just raw numbers
- **Pricing section** with per-token cost and real-world daily/monthly estimates
- **Risk section** with honest caveats — don't sugarcoat
- **Strategic implications** section tailored to the user's role and toolchain
- **External reference links** section with all URLs used

---

## Phase 6: Obsidian Knowledge Base Integration

Save to the user's Obsidian vault with proper frontmatter:

```markdown
---
title: [Provider Name] 全面深度研究報告
tags:
  - AI/ [Provider]
  - research/ deep-dive
  - [relevant tags]
created: [YYYY-MM-DD]
source: [primary source URL]
---

[Full report content]
```

**Vault path:** `C:\Users\ysga1\Documents\Lunian\知識庫\[filename].md`

**Frontmatter rules:**
- `title`: Provider name + "全面深度研究報告"
- `tags`: At minimum `AI/ [Provider]` and `research/ deep-dive`
- `created`: current date in `YYYY-MM-DD` format
- `source`: primary homepage URL

---

## Phase 7: Multi-Provider Routing Analysis (Extension)

When the user asks to combine multiple providers, extend the report with a **routing analysis section**:

1. **Side-by-side comparison table** — specs, benchmarks, pricing, capabilities
2. **Routing strategy designs:**
   - **Tiered Stack**: 70% cheap/free + 25% mid + 5% premium
   - **Specialist Routing**: each model does what it does best
   - **Fallback Consensus**: cheap first, fallback on failure
   - **Cache Optimization**: stable prefixes on provider with cache discounts
3. **Cost projections**: $/month at various traffic levels
4. **Risk matrix**: provider-specific risks (policy change, deprecation, geopolitical)
5. **Implementation sketch**: Python router class with classification logic

---

## Pitfalls

- **Don't trust official benchmarks alone** — always cross-check with independent evaluators (Claw-Eval, Artificial Analysis, SWE-bench, etc.)
- **Don't conflate Pass³ with Pass@3** — they measure different things (Pass³ = % tasks passed in 3 runs, Pass@3 = % tasks passed at least once in 3 attempts)
- **Don't ignore date** — AI landscape changes weekly; a 3-month-old article is outdated
- **Don't present speculative articles as facts** — label projections clearly
- **Don't present one source's pricing without verifying** — check the actual API docs
- **Don't stop at the homepage** — the most useful info is in docs, blog posts, and third-party analysis
- **Cache hit pricing matters** — DeepSeek gives 50-120x discounts; ignoring this underestimates cost effectiveness
- **Harness discrepancies are real** — same model can score 80% on one SWE-bench harness and 8% on another. Understand the harness, not just the number

---

## Cross-References

- `project-architecture-audit` — Deep-dive methodology for codebase/project analysis (complementary: codebases vs providers)
- `hermes-custom-providers` — Setting up the researched provider in Hermes config
- `hermes-custom-providers` → `references/agnes-ai-integration.md` — Agnes AI provider reference

## Verification Checklist

- [ ] All claims cross-validated across 3+ independent sources
- [ ] Pricing verified against actual API docs (not just blog posts)
- [ ] Benchmarks annotated with source and date
- [ ] Discrepancies between official and independent data flagged
- [ ] Risk section includes honest caveats
- [ ] Strategic implications tailored to this user (prompt engineer, agent developer)
- [ ] Report saved to Obsidian with correct frontmatter
- [ ] External reference links section included
