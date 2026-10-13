# Research: response density, pacing, and adaptation for short-attention chat mode

Date: 2026-10-13
Method: three parallel background researchers, primary sources only (official specs/docs, original papers, standards, style guides). Findings consolidated from their reports; every claim cites its source.

## The design problem

The `adhd` skill caps chat responses at ~250 tokens in Point→Evidence→Explanation order. It shortens replies, but on hard topics the agent crams too many **new concepts** into one dense paragraph. Working hypotheses under test:

1. The right budget is *new concepts per response* (density), not *tokens* (length).
2. A hard topic should take **more short turns**, not longer turns.
3. Explanations should adapt to the user's skill level.
4. Concrete instances should come before general definitions.
5. The purpose of short turns is preserving back-and-forth: the user must still remember their own question when they finish reading.

---

## 1. How existing AI products handle it

### 1.1 Model specs and published system prompts

**OpenAI Model Spec** (2026-08-18, [model-spec.openai.com](https://model-spec.openai.com/2026-08-18.html)) sets no numeric length target; it states competing considerations. "Be thorough but efficient, while respecting length limits": thoroughness favors "immediately usable" informative responses; brevity is justified by avoiding "uninformative or redundant text" and interruptions at output limits. Crucially for our bug: "**Adapt length and structure to user objectives** — assistant voice responses should align with the iterative, turn-taking structure of spoken conversation." The spec's own *violation example* for this rule is exactly our failure mode: a correct but dense multi-clause paragraph where a ~3-sentence direct answer was wanted. Also: "If the user asks a question, the response should be phrased as a direct answer rather than a list of facts," and formatting should be used "judiciously to aid the user in scanning."

**Claude Code "Concise" output style** ([code.claude.com/docs/en/output-styles](https://code.claude.com/docs/en/output-styles)): "the first sentence of a response states what happened or what the answer is. Claude leaves out the lead-in, the step-by-step narration, and the closing recap, and answers a simple question in one to three sentences." Full-length exceptions: "Anything you ask for" and "Anything you need in order to act safely" (error reports, failing test output, security warnings, destructive-action confirmations). Pattern: cap per-turn prose, exempt load-bearing content, never compress the work.

**Gemini CLI** (system prompt in open source, [snippets.ts](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/prompts/snippets.ts)): "Minimal Output: Aim for fewer than 3 lines of text output … per response whenever practical"; "No Chitchat"; "No Repetition"; "For simple or direct requests, prioritize extreme brevity."

### 1.2 Tutoring products ration the answer across turns

**Khanmigo** ([khanmigo.ai/learners](https://www.khanmigo.ai/learners)): "doesn't just give answers … guides learners to find the answer themselves"; "never giving you the answer"; "will walk you through the solution and ask questions so you can work it out yourself." Turn size stays small because the *answer itself* is rationed across many turns.

**Duolingo Max** ([blog.duolingo.com/duolingo-max](https://blog.duolingo.com/duolingo-max/)): chat explanations are deliberately "a simple explanation" of one exercise, with clarification pulled by the learner. Roleplay turns are scripted first-party: "Our experts also write the initial message in the chat and tell the model where to take the conversation" — turn structure is designed, not left to the model.

### 1.3 Long-running agent products: many short, announced messages

- **Gemini CLI "Topic Updates"**: "the user follows along by reading topic updates that you publish … A topic is typically a discrete subgoal and will be every 3 to 10 turns." Plus "Explain Before Acting: Never call tools in silence."
- **Devin** ([docs.devin.ai](https://docs.devin.ai/essential-guidelines/instructing-devin-effectively.md)): "Breaking down complex tasks into smaller checkpoints … Ask Devin to report back after completing each checkpoint"; "share a short plan before making changes."
- **Claude Code** ([best practices](https://code.claude.com/docs/en/best-practices)): plan mode separates exploration from execution; Anthropic describes it as "keeping you in the loop at every step" ([3.7 launch](https://www.anthropic.com/news/claude-3-7-sonnet)).

### 1.4 Length is a biased metric

- **Length-Controlled AlpacaEval** (Dou et al., [arXiv:2404.04475](https://arxiv.org/abs/2404.04475)): benchmarks "favor models that generate longer outputs"; length-controlling raised correlation with human preferences from 0.94 to 0.98.
- **LMSYS style control** ([lmsys.org](https://lmsys.org/blog/2024-08-28-style-control/)): controlling for "answer length, markdown header count, markdown bold count, and markdown list element count" changed leaderboard rankings materially.
- **Reward-model overoptimization** (Coste et al., [arXiv:2312.09244](https://arxiv.org/abs/2312.09244)): RLHF amplifies small reward-model quirks; length pressure follows whichever metric is rewarded.
- First-party verbosity controls are **user-set session modes**, not per-reply guesses: GPT-5 `verbosity` parameter (low = "terse UX, minimal prose"; high = "great for audits, teaching, or hand-offs", [openai-cookbook](https://github.com/openai/openai-cookbook/blob/main/examples/gpt-5/gpt-5_new_params_and_tools.ipynb)); Claude Code output styles switched per session.

### 1.5 Progressive disclosure, first-party

Anthropic's Agent Skills engineering post ([anthropic.com/engineering](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)): "Progressive disclosure is the core design principle" — one-line metadata loaded first, body loaded on demand, details moved to separate files "to keep the core … lean." The same layering (headline now, depth on demand) maps to chat turns.

**Product pattern in one line:** every primary source caps *per-turn prose*, exempts *load-bearing content*, and moves bulk across *many announced turns* — none compresses the whole topic into one turn.

---

## 2. What educational theory says

### 2.1 Working memory: the budget is chunks, and chunks are knowledge-relative

- **Miller (1956)**, *Psychological Review* 63:81–97 ([text](http://psychclassics.yorku.ca/Miller/)): immediate memory span ≈ 7 items, but "we're not very definite about what constitutes a chunk"; capacity is a fixed number of chunks, so information held depends on chunk size (recoding).
- **Cowan (2001)**, *Behavioral and Brain Sciences* 24:87–114 ([DOI](https://doi.org/10.1017/S0140525X01003922)): the real limit is "a single, central capacity limit averaging about four chunks" (3–5). An expert's chunk can be an entire concept, so the same paragraph is 1 chunk to an expert and 6 to a novice.

### 2.2 Cognitive Load Theory: element interactivity is the formal "information density"

**Sweller, van Merriënboer & Paas (2019)**, *Educational Psychology Review* 31:261–292 ([DOI](https://doi.org/10.1007/s10648-019-09465-5)):

- Intrinsic load = "the complexity of the information being processed," related to **element interactivity** — the number of elements that must be held in mind simultaneously.
- Complexity "depends on a combination of both the nature of the information **and the knowledge of the person processing the information**" — so "measures that ignore knowledge when determining complexity are largely useless."
- Learning compresses density: "Concepts and procedures that consisted of multiple elements can, with increases in expertise, be stored in long-term memory as a single element."
- Approved remedy for high intrinsic load: the **isolated→interacted elements effect** — "presenting the individual elements" first, "followed by all of the information including … their interactions," and simple-to-complex sequencing.

This is the theoretical grounding for a concept budget: the load-bearing variable is the number of simultaneously held *unfamiliar* elements, not token count.

### 2.3 Mayer's principles — the evidence for splitting and ordering

From Mayer & Moreno (2003), *Educational Psychologist* 38(1):43–52 ([PDF](https://www.uky.edu/~gmswan3/544/9_ways_to_reduce_CL.pdf)) and Mayer, *Multimedia Learning* (2009):

- **Segmenting**: "Students understand a multimedia explanation better when it is presented in learner-controlled segments rather than as a continuous presentation." Effect size 1.36 on transfer (Mayer & Chandler 2001). This is the remedy when essential processing exceeds capacity — i.e., hard topics.
- **Pretraining**: "Provide pretraining in names and characteristics of components" — teach the nouns before the causal story. Median effect size 1.00 (Mayer, Mathias & Wetzell 2002).
- **Coherence**: excluding "interesting but extraneous material" improves understanding.
- **Signaling**: cues about how to process the material improve understanding.

### 2.4 Expertise reversal — the argument for user-skill adaptation

**Kalyuga et al. (2003)**, *Educational Psychologist* 38(1):23–31 ([DOI](https://doi.org/10.1207/S15326985EP3801_4)): "Instructional techniques that are highly effective with inexperienced learners can lose their effectiveness and even have negative consequences when used with more experienced learners." Kalyuga (2007), *Educational Psychology Review* 19:509–539, reviews this as the case for "learner-tailored instructional systems." Mechanism (Sweller et al. 2019): guidance designed for novices becomes redundant load as expertise grows. Combined with §2.2, density settings that ignore the user's level are theoretically unjustifiable.

### 2.5 Scaffolding and fading

- **Wood, Bruner & Ross (1976)**, *J. Child Psychology and Psychiatry* 17:89–100 ([DOI](https://doi.org/10.1111/j.1469-7610.1976.tb00381.x)): scaffolding = the tutor "controlling those elements of the task that are initially beyond the learner's capacity"; key function "reduction in degrees of freedom" — simplify by cutting the number of constituent acts.
- **Renkl & Atkinson (2003)**, *Educational Psychologist* 38(1):15–22 ([DOI](https://doi.org/10.1207/S15326985EP3801_3)): fade worked examples into problem solving stepwise, "adaptive to the learners' prior knowledge levels." Support is withdrawn on demonstrated success, not on a schedule.

### 2.6 What effective human tutors do turn-by-turn

- **Chi et al. (2001)**, *Cognitive Science* 25:471–533 ([DOI](https://doi.org/10.1207/s15516709cog2504_1)): students learned as well when tutors were "suppressed from giving explanations" and instead prompted — effective tutoring **elicits** rather than lectures.
- **Chi et al. (1994)**, *Cognitive Science* 18:439–477: prompting self-explanations of each step yields large gains in deep understanding.
- **Chi & Wylie (2014)**, ICAP framework (*Educational Psychologist* 49:219–243): outcomes increase monotonically Interactive > Constructive > Active > Passive.

### 2.7 Concreteness fading — concrete before general

**Fyfe, McNeil, Son & Goldstone (2014)**, *Educational Psychology Review* 26:9–25 ([DOI](https://doi.org/10.1007/s10648-014-9249-3)): effective sequences start concrete, move to representational, end abstract-symbolic (origins: Bruner 1966, enactive→iconic→symbolic). Concrete referents make abstract symbols interpretable and expose generalizable structure. **McNeil & Fyfe (2012)**, *Learning and Instruction* 22:449–455: fading promotes transfer better than starting abstract. This validates the ACVF-table observation: concrete cases first; the general definition should arrive *after*, as compression of the cases.

### 2.8 Given–New contract — one new element per assertion

**Clark & Haviland (1977)**, "Comprehension and the Given-New Contract" ([PDF](http://www.web.stanford.edu/~clark/1970s/Clark,%20H.H.%20_%20Haviland,%20S.E.%20_Comprehension%20and%20the%20given-new%20contract_%201977.pdf)): "speakers and listeners have an implicit agreement about how (a) information that is known … and (b) information that is novel … are to appear in sentences." Listeners find the antecedent for the Given, then attach the New; utterances with too much New or unknown Given are comprehended slower and judged inappropriate. (Origins: Halliday 1967, *J. Linguistics* 3:199–244.) One turn ≈ one new element anchored to established ground.

### 2.9 Retrieval practice — end turns with a check, not a recap

- **Roediger & Karpicke (2006)**, *Psychological Science* 17:249–255 ([DOI](https://doi.org/10.1111/j.1467-9280.2006.01693.x)): "Testing is a powerful means of improving learning, not just assessing it."
- **Bisra et al. (2018)** meta-analysis, *Educational Psychology Review* 30:703–725: self-explanation prompts g ≈ 0.55.
- Constraint: **Rittle-Johnson & Loehr (2016)**, *Psychonomic Bulletin & Review* 23:550–557 — benefits shrink with high prior knowledge (expertise reversal again).

A short check at the turn boundary is the highest-yield use of the boundary — and doubles as the signal for fading and density control.

---

## 3. What communication professions prescribe

### 3.1 Plain language

- **US Federal Plain Language Guidelines** ([digital.gov/guides/plain-language](https://digital.gov/guides/plain-language); source [GitHub](https://raw.githubusercontent.com/gsa/plainlanguage.gov/main/_pages/guidelines/audience/index.md)): "The first rule of plain language" is write for your audience — "Know the expertise and interest of your average reader"; answer before writing: who is my audience, what do they already know, what do they need to know, what will they ask? Organization: "Start by stating your purpose and the bottom line"; most important information first, background last; "Start with your main idea – not an exception."
- **ISO 24495-1:2023** ([iso.org](https://www.iso.org/standard/78907.html)): plain language = readers can **find**, **understand**, and **use** what they need, grounded in relevance to the reader's knowledge and context.

### 3.2 Inverted pyramid and BLUF

Poynter ([Scanlan 2003](https://www.poynter.org/reporting-editing/2003/writing-from-the-top-down-pros-and-cons-of-the-inverted-pyramid/)): "The inverted pyramid puts the most newsworthy information at the top, and then the remaining information follows in order of importance" — invented so editors "could cut stories from the bottom up: no news judgment required." **BLUF** (Army Regulation 25-50): "the main point at the beginning of the correspondence." Front-loading makes truncation safe at any point — the same property that makes a long answer safe to abandon mid-read.

### 3.3 Carroll's minimalism

van der Meij & Carroll, in *Minimalism Beyond the Nurnberg Funnel* (MIT Press, 1998; [DOI](https://doi.org/10.7551/mitpress/4616.003.0003)): (1) action-oriented — organize around the user's real task, not the system's feature list; (2) support error recognition and recovery rather than preventing everything; (3) support reading to do/study/locate — "read less, do more," text skimmable and skippable; (4) anchor in what the learner already knows.

### 3.4 Diátaxis — one mode per response

[diataxis.fr](https://diataxis.fr/): four distinct needs → tutorials, how-to, reference, explanation. Mixing is explicitly harmful: a how-to should contain "action and only action — no digression, explanation, teaching"; explanation "tends to absorb other things," which "interferes with the explanation itself." A chat reply should detect the user's mode and serve only that mode, deferring the rest.

### 3.5 Nielsen Norman Group — the UX numbers

- **Progressive disclosure** ([nngroup.com](https://www.nngroup.com/articles/progressive-disclosure/)): "defers advanced or rarely used features to a secondary screen, making applications easier to learn and less error-prone." "Limit to 2 disclosure levels" — users "often get lost" beyond two.
- **Chunking** ([nngroup.com](https://www.nngroup.com/articles/chunking/)): "Don't ask your users to hold more than a few pieces of information in their short-term memory at once"; "the size of the chunks did not seem to matter" — meaning, not word count, defines a chunk. (Chunking matters least when nothing must be retained.)
- **Scanning** ([how users read](https://www.nngroup.com/articles/how-users-read-on-the-web/)): 79% of test users scanned; only 16% read word-by-word. Concise writing = 58% better usability; scannable = 47%; combined = 124%. Users read "at most 28% of the words … 20% is more likely" ([how little](https://www.nngroup.com/articles/how-little-do-users-read/)).

### 3.6 Health communication: chunk-and-check / teach-back

**AHRQ Health Literacy Toolkit, Tool #5** ([ahrq.gov](https://www.ahrq.gov/health-literacy/improve/precautions/tool5.html)): teach-back = "a way of checking understanding by asking patients to state in their own words what they need to know or do"; "based on 'chunk and check' — explain one chunk, then check for understanding, then explain the next chunk." **CDC Clear Communication Index** ([cdc.gov/ccindex](https://www.cdc.gov/ccindex/)): "Make sure the material has one main message statement"; it "must be in the first paragraph or section." This is a production protocol for exactly the back-and-forth pacing the skill wants.

### 3.7 Motivational interviewing: short reflective turns

Miller & Rollnick 2013, via [MINT](https://motivationalinterviewing.org/understanding-motivational-interviewing): OARS — open questions, affirmations, reflections, summaries. Reflections ("repeating, rephrasing or offering a deeper guess") are the primary empathy move; short reflective turns elicit the person's own reasoning instead of one long persuasive monologue.

### 3.8 Conversation analysis: turn-taking norms

**Sacks, Schegloff & Jefferson (1974)**, *Language* 50(4):696–735 ([full text](https://www.conversationanalysis.org/mp-files/simplest-systematics-for-the-organization-of-turn-taking-for-conversation-language.pdf/)):

- "Overwhelmingly, one party talks at a time"; "Turn size is not fixed, but varies"; "there will be pressure for minimization of turn size."
- **Adjacency pairs**: a question "makes answer specially relevant for next turn" — one first pair-part allocates exactly one next turn; stacked questions force selection and forgetting.
- **Recipient design** (§5): talk is "constructed or designed in ways which display an orientation and sensitivity to the particular other(s)" — audience adaptation is constitutive of natural turns, not polish.
- Problems of understanding "are preferably raised" immediately after the turn — the natural home for a check is the turn boundary.

### 3.9 Style guides

- **Google developer style guide** ([paragraph structure](https://developers.google.com/style/paragraph-structure)): "Each paragraph should address a single idea in the fewest words"; "A paragraph longer than 5 or 6 sentences is often an indication that the paragraph is trying to convey too much information."
- **Microsoft Writing Style Guide** ([top 10 tips](https://learn.microsoft.com/en-us/style-guide/top-10-tips-style-voice)): "Get to the point fast. Lead with what's most important"; "Three to seven lines is about the right length for a paragraph."

---

## 4. Synthesis — what the evidence supports for the skill redesign

| Design decision | Evidence |
| --- | --- |
| **Primary control = new concepts per turn, not tokens** | CLT element interactivity is the formal density measure and is knowledge-relative (Sweller 2019); chunk size doesn't matter, chunk *count* does (NN/g); the Model Spec's violation example is a dense paragraph; Google "single idea" per paragraph; CDC "one main message statement." |
| **Hard topic → more turns, not longer turns** | Segmenting principle, effect size 1.36 (Mayer); isolated→interacted elements (Sweller 2019); chunk-and-check (AHRQ); topic updates every 3–10 turns (Gemini CLI); checkpoints (Devin). |
| **Exemptions: never compress load-bearing content** | Claude Code Concise exempts what the user asked for and what safety requires; OpenAI spec avoids truncating at output limits; artifacts already exempt in our skill. |
| **Concrete instance before general definition** | Concreteness fading (Fyfe 2014; McNeil & Fyfe 2012); pretraining — nouns before causal story (Mayer, ES ≈ 1.00). |
| **Adapt to user skill** | Expertise reversal (Kalyuga 2003/2007); Cowan — chunks are knowledge-relative; plain language "know your reader" four questions; recipient design (SSJ 1974); first-party products do it with session-level modes, not per-reply guesses. |
| **Anchor each turn on Given, add one New** | Given–New contract (Clark & Haviland 1977). |
| **End hard-topic turns with a check or a named next piece, not a recap** | Retrieval practice (Roediger & Karpicke 2006); ICAP (Chi & Wylie 2014); adjacency pairs — one question allocates one next turn (SSJ 1974); AHRQ script. NOTE: current skill's "no offers of follow-up work" bans scope-creep offers, which is different from a next-piece hook — needs an explicit carve-out. |
| **Fade by demonstrated success** | Renkl & Atkinson 2003; Wood/Bruner/Ross contingency. |
| **Lead with the answer (truncation-safe order)** | Inverted pyramid/BLUF; NN/g; Claude Code "first sentence states the answer" — our PEE Point-first is already right; the failure is density, not order. |
| **Skimmability over brevity theater** | Length is a biased, gameable metric (AlpacaEval LC; LMSYS style control); concise + scannable beat merely short (NN/g 124%). |

### Open tensions to resolve when revising SKILL.md

1. "No offers of follow-up work" vs. the evidence that a turn-final hook is the mechanism that preserves the loop. Carve out: a *named next piece of the same topic* is allowed on multi-turn explanations; scope-creep offers remain banned.
2. Token ceiling: keep as backstop (products do keep per-turn caps) but demote it below the concept budget; define exemptions.
3. Skill assessment: evidence favors a cheap, session-level calibration (mirror the user's vocabulary; ask at most once) over per-reply inference machinery.
4. Tables/concrete cases: NN/g notes users scan and skip; a small concrete-case table is the evidence-backed format for multi-object answers (and matches the user's ACVF-table experience).
