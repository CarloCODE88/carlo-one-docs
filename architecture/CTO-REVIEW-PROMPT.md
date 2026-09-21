# CTO Review Prompt

You are the CTO reviewing the CarloONE Fusion Foundation Plan.

Please review the plan at:

- `architecture/FUSION-FOUNDATION-PLAN.md`

Use the following documents as verification input:

- `architecture/TECHNICAL-BASELINE.md`
- `architecture/TRIAI-ENGINE-ARCHITECTURE.md`
- `operations/HIXX-SERVER-ARCHITECTURE-AND-OPERATIONS.md`
- `operations/TRI-HIXX-DATA-TREE.md`
- `guides/CTO-FUSION-PLAN.md` as legacy strategic intent only, not as verified execution truth

Your job is to judge whether the foundation plan is:

- technically defensible
- operationally realistic
- properly prioritized
- aligned to the actual source state
- appropriately strict about boundaries and decision gates

Please answer in the following structure:

1. Verdict
   - approve / approve with changes / reject
2. What is strong in the plan
3. What is weak or risky
4. Missing decisions that must be made before implementation
5. Changes you would require before approving execution
6. Final recommendation for the team

Specific review criteria:

- Does the plan properly prioritize triAI as the productive backend path?
- Does it correctly treat HIXX as a prototype or sidecar until ABI and operational boundaries are decided?
- Does it define the correct architecture gates before product or premium expansion?
- Does it avoid overclaiming maturity or fixed timelines?
- Does it distinguish clearly between verified facts, proposed changes, and human decisions?
- Does it retain only the useful parts of the legacy CTO fusion document without treating it as execution truth?

If you disagree with any part of the plan, explain exactly why and what should change.

If you agree, explain under which conditions the plan should proceed and which decisions must be formalized before implementation starts.

Please be concrete, concise, and actionable.
