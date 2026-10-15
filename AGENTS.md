# Cloudflare Artifacts skill maintenance

Keep this repository private. The user will make it public when ready.

The skill is `cloudflare-artifacts`, located at `skills/cloudflare-artifacts/`;
`-skill` belongs only to the repository name. Keep automatic skill discovery
enabled.

Artifacts is in beta. Before changing a claim, check the live docs (any page
as Markdown: append `index.md`; index at
https://developers.cloudflare.com/artifacts/llms.txt), the `Artifacts`
declarations in the latest `@cloudflare/workers-types`, and Wrangler's
`config-schema.json` and CLI source. Live docs beat old notes; where the docs
contradict themselves or the tooling, say so in
`references/limits-pricing-errors.md` rather than picking silently. Update the
"Verified against" line in `SKILL.md` with the date and versions.

Write examples from scratch; do not paste Cloudflare documentation text.
Never put a real account ID, API token or repo token in this repository.
Type-check TypeScript examples against `@cloudflare/workers-types` and compile
Go examples in an isolated temporary module before committing.
