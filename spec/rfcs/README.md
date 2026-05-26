# Polaris RFC Process

All non-trivial changes to the Polaris protocol — contract modifications, parameter changes, new data services, oracle design — require an RFC (Request for Comments) before implementation begins.

## When is an RFC required?

- Any change to contract semantics (not just a constant or address)
- Any new contract
- Any change to the reward distribution formula
- Any change to the oracle eligibility criteria
- Any addition to the payment token whitelist
- Any change to on-chain governance mechanics

Not required:
- Typo fixes in documentation
- Test additions
- Deployment scripts
- Tooling changes

## Lifecycle

```
Draft → Under Review → Accepted → Implemented → Withdrawn
```

| Status | Meaning |
|--------|---------|
| Draft | Author is writing; not ready for broad review |
| Under Review | MR open; community commenting |
| Accepted | MR merged to `main`; implementation may proceed |
| Implemented | Corresponding GitLab issue closed; code on `main` |
| Withdrawn | Superseded or abandoned; not implemented |

## How to write an RFC

1. Copy `spec/rfcs/0000-template.md` to `spec/rfcs/NNNN-{slug}.md` where `NNNN` is the next available four-digit number.
2. Fill in all sections. Incomplete RFCs will not be merged.
3. Open an MR with title `RFC NNNN: {title}`.
4. Link the MR to the corresponding GitLab issue.
5. Allow at least **7 days** of open review before merging.

## RFC and issue relationship

Each GitLab issue in the `rfc::required` state is blocked on its RFC being merged to `main`. The issue description links to the RFC file. The RFC links back to the issue. When the RFC is accepted (MR merged), the issue moves to `rfc::accepted` and implementation may begin.

Implementations are MRs against `contracts/` or `spec/` that close the issue. They must reference the accepted RFC in their description.
