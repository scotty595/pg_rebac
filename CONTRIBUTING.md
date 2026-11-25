# Contributing

Thank you for wanting to improve pg_rebac. It is small on purpose: one file of SQL (`sql/rebac.sql`), one daemon
(`crates/rebacd`) and the tests and studies that hold them to their promises. A change is welcome when it keeps them
that way.

## Find something to work on

Issues labelled [`good first issue`](https://github.com/louvain-ai/pg_rebac/labels/good%20first%20issue) are scoped to
a day or less and say where to start. Before a larger change (a new schema shape, a new mode, a change to what a read
returns), open an issue first so we can agree on the shape before you write it.

## Run it

```sh
docker compose -f examples/quickstart/docker-compose.yml run --rm demo   # see it work, end to end
docker compose up -d --wait                                              # the test stack: Postgres on :55433, SpiceDB on :50061
```

## Before you open a pull request

Every one of these must pass; they are what a reviewer runs.

```sh
REBAC_TEST_URL=postgres://postgres:rebac@127.0.0.1:55433 tests/sql/run.sh    # the SQL suite
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
REBAC_IT=1 cargo test -p rebacd --test integration                           # the daemon against a real SpiceDB
```

- **A change to what a read returns ships with a test that fails without it.** The SQL tests are plain `psql` scripts
  with assertions that raise (`tests/sql/_helpers.sql`); the daemon's are in `crates/rebacd/tests`.
- **The fence fails closed.** A change must never turn an error (`RB400`, `RB404`, `RB503`) into an empty set, or an
  empty set into rows. If a change could, say how the tests show it does not.
- **A claim about speed comes with its measurement.** `docs/planner.md` and `examples/hybrid-pgvector/` show the
  shape: the set-up, the numbers and the script that reproduces them.
- **Docs move with the code.** Update `README.md`, `docs/` and the site (`site/src/content/docs/`) in the same pull
  request when behaviour, a guarantee or a limit changes. `cd site && npm ci && npm run build` must pass.
- Keep a pull request to one concern, and say in its description what it changes and how you checked it.
- Never include credentials, real personal data or a real customer's text in code, tests or fixtures. Use invented
  names.

## Commit messages

Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): subject`, where the scope
is optional (`rebacd`, `sql`, `schema`, `npm`, `quickstart`, `site`, `ci`). The type decides the next version and
whether the commit appears in the CHANGELOG, so write a `feat` or `fix` subject as the line a reader of the release
notes should see:

| Type | Use it for | Next version (before 1.0) | In the CHANGELOG |
|---|---|---|---|
| `feat` | something a user can now do | minor | Features |
| `fix` | something that was wrong for a user | patch | Fixes |
| `perf` | the same result, faster | patch | Performance |
| `docs`, `test`, `refactor`, `build`, `ci`, `chore` | everything else | none | no |

A change that breaks an install, a configuration or what a read returns adds `!` after the type (`feat!:`) and a
`BREAKING CHANGE:` footer saying what to do; before 1.0 it bumps the minor version. A pull request's title follows
the same form, because a squash merge makes it the commit. CI checks both (`commitlint`, configured in
`.commitlintrc.json`). Don't add version headings to `CHANGELOG.md` by hand; release-please writes them.

## Releasing

A release is one act: merging the release pull request.

- **The release pull request.** On every push to `main`, release-please (`.github/workflows/release-please.yml`,
  `release-please-config.json`) opens or updates a pull request titled `chore(main): release X.Y.Z`. It bumps every
  version (the Cargo workspace, `rebacd`'s dependency on `rebac-schema`, `Cargo.lock`, `META.json`,
  `packages/npm/package.json`), writes the CHANGELOG entry from the commits since the last release, and points the
  site's one-file quickstart (`site/public/quickstart.yml`) at the new version. To change what it says, change the
  commits, not the pull request: release-please rewrites it on every push.
- **Merging it** tags `vX.Y.Z`, creates the GitHub Release, and starts `.github/workflows/publish.yml`. That
  workflow checks the tag against every manifest and runs the SQL suite, then publishes crates.io (`rebac-schema`,
  then `rebacd`); then npm (with provenance) and, beside it, the `ghcr.io/louvain-ai/rebacd` image (linux/amd64 and
  linux/arm64, with an SBOM and provenance, signed with cosign); then the Release's assets (the SQL, `SHA256SUMS`,
  the PGXN zip) and PGXN. Last, it runs the served quickstart against what it published and deploys pgrebac.com
  from the tag's tree, so the site never shows unreleased docs. Every step skips what is already published, so a
  failed run is fixed by running it again; an npm failure holds up nothing else.
- **A dry run** builds and checks everything and publishes nothing:
  `gh workflow run publish.yml --ref vX.Y.Z -f tag=vX.Y.Z` (`dry-run` defaults to true).
- **A patch for an older minor version** is released from a maintenance branch `release-X.Y`, cut from the last
  `vX.Y.Z` tag, with this setup on it, `"versioning": "always-bump-patch"` in its `release-please-config.json` and
  `.release-please-manifest.json` set to `X.Y.Z`. Fixes are cherry-picked onto it, and release-please keeps a
  release pull request there too. Unless it is the highest version, its release moves neither npm's `latest`
  dist-tag, the image's `latest` tag, the GitHub Release marked latest nor the site.
- **The manual fallback**, when GitHub Actions cannot run the release: `scripts/release.sh X.Y.Z` from the release
  commit. It asks before each step and skips what is published; run it with `--dry-run` first.

To verify a release image:

```sh
cosign verify ghcr.io/louvain-ai/rebacd:X.Y.Z --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github\.com/louvain-ai/pg_rebac/\.github/workflows/publish\.yml@refs/tags/v'
```

## Sign your commits (DCO)

Every commit must be signed off under the Developer Certificate of Origin (<https://developercertificate.org/>): by
signing off you certify that you wrote the change, or otherwise have the right to submit it under this repository's
licence. `git commit -s` adds the line:

```
Signed-off-by: Your Name <you@example.com>
```

## Security

Do not open a public issue for a vulnerability. See [SECURITY.md](SECURITY.md).

## Conduct

Everyone taking part is expected to follow the [code of conduct](CODE_OF_CONDUCT.md).
