# adhd

A response-style skill governing how the agent writes conversational replies: short, direct, PEE-structured. Source of truth lives here; `SKILL.md` is copied to `~/.agents/skills/adhd/` for activation.

## Language

### Request types

**Inquiry**:
A user-initiated request seeking an answer that closes a current knowledge gap.
_Avoid_: question, ask

**Status report**:
An agent-initiated account of the outcome of requested work; consumed in the conversation.
_Avoid_: update, progress report

**Clarification**:
An agent-initiated question asking for missing information. Confirmation — a consent request naming a single action — is a subtype, not a separate type.
_Avoid_: disambiguation, follow-up question

**Other request**:
Any request whose response is produced for reuse rather than consumed in the conversation (documentation, reports, plans, code artifacts). The adhd skill has no authority here.
_Avoid_: task request, deliverable

### Response format

**PEE**:
The governed response shape — Point, then Evidence, then Explanation, as plain prose. Collapses to Point alone when the answer is self-evident.
_Avoid_: essay format, structured answer

**Necessary elaboration**:
The high-level Evidence and Explanation required to support the Point — zero when the Point stands alone.
_Avoid_: detail, context

**Budget**:
The 250-token ceiling on agent prose per governed response; a target, not a literal count. Artifacts are exempt.
_Avoid_: limit, cap
