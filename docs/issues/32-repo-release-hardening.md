# Title

Repository & release hardening (GitHub settings, release workflow)

## Summary

Prepare the repository for public OSS release: security-relevant GitHub
settings (some maintainer-manual), CodeQL + secret scanning, the tag-driven
release workflow with PyPI Trusted Publishing (OIDC), release checklist, and
the v0.1.0 release-readiness gate.

## Context

DESIGN §8 T8/T9 and the ISSUE_PLAN release gate. Split of duties: the
implementation agent authors workflows/files and a **maintainer runbook**;
org/repo settings and any credential/trusted-publisher registration are
executed manually by the maintainer (never by an agent — house rule).

## Scope

- `.github/workflows/codeql.yml`, `release.yml`
- `.github/CODEOWNERS`, `CONTRIBUTING.md` (finalize), issue/PR templates
- `docs/RELEASING.md` (runbook + checklist)

## Detailed Requirements

1. `codeql.yml`: python, on PR + weekly schedule; SHA-pinned;
   `permissions: security-events: write, contents: read`.
2. `release.yml`: trigger `push: tags: v*`;
   jobs: build (`uv build`, artifacts uploaded), publish to **PyPI via
   Trusted Publishing** (`pypa/gh-action-pypi-publish` SHA-pinned,
   `environment: release` with required reviewers, `permissions:
   id-token: write, contents: read`), GitHub Release creation with
   auto-generated notes + `CHANGELOG.md` excerpt. **The publish job is
   gated on the `release` environment approval — a human click.**
   No secrets stored; OIDC only.
3. `CHANGELOG.md`: Keep-a-Changelog format, seeded with 0.1.0 section
   collecting the wave summaries.
4. `CODEOWNERS`: `* @Saber5656` (default owner; adjust if teams appear).
5. Issue templates: bug (with `openkaraoke doctor --json` attachment ask +
   privacy warning about paths in logs), feature request; PR template with
   the security checklist line ("touched input parsing / subprocess /
   network? cite the DESIGN §8 row").
6. `docs/RELEASING.md` runbook:
   - Maintainer-manual settings checklist (verify current state, document
     evidence screenshots/URLs): default-branch ruleset (already: PR-only,
     no force push — verify), secret scanning + push protection ON,
     Dependabot alerts ON, private vulnerability reporting ON, Actions
     default token read-only, release environment with required reviewer,
     PyPI Trusted Publisher registration steps (exact project/workflow
     names).
   - Release checklist (from ISSUE_PLAN validation item 5): full CI green,
     e2e green both OSes, `models verify` on a fresh cache, TestPyPI
     rehearsal (30), one real-song manual validation per language
     (21/31 playbooks), SECURITY/NOTICE/README review, version bump PR,
     tag, approve environment, verify PyPI artifact hash equals CI
     artifact.
7. Verify (and document in the PR) that no workflow uses unpinned actions,
   `pull_request_target`, or write-scope defaults — extend the actionlint
   step of issue 02 into CI here (`actionlint` job, SHA-pinned binary).

## Acceptance Criteria

- [ ] CodeQL runs green; actionlint job green.
- [ ] `release.yml` dry-run: tag `v0.1.0-rc1` on a branch → build job
      produces artifacts; publish job halts awaiting environment approval
      (do **not** approve to production PyPI during this issue; TestPyPI
      target used for the rehearsal via a temporary workflow input —
      document and remove).
- [ ] RELEASING.md manual-settings checklist completed by the maintainer
      with evidence links (issue comment).
- [ ] Templates/CODEOWNERS/CHANGELOG present and lint-clean.
- [ ] Grep audit: zero unpinned `uses:` across all workflows.

## Validation

```bash
actionlint .github/workflows/*.yml
gh workflow run codeql.yml && gh run watch
git tag v0.1.0-rc1 <sha> && git push origin v0.1.0-rc1   # rehearsal per runbook
```

## Dependencies

- 02 (CI base), 30 (build artifacts correct)

## Non-goals

- Actual v0.1.0 production publish (maintainer decision after the release
  gate); artifact signing/SLSA provenance (v2 hardening).

## Design References

- DESIGN §8 T8/T9, §11; ISSUE_PLAN validation item 5; ADR-006
