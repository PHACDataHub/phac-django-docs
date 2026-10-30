## Python dependency version constraints

There's a tradeoff between reliability (tightly pinning versions, fewer breaking bugs) and security (allowing security upgrades from latest versions). If a security patch gets screened in by versioning criteria rules, we often don't need to do anything (at most: just clicking a deployment button). If it doesn't, we need a pull-request to upgrade that dependency's criteria in `requirements.txt`, which is more work. 

Choose dependency constraints based on the package's documented compatibility guarantees and release history.

- **The ideal: major-version bounds**
  - `>=1.2.3,<2`
  - Use for packages that follow Semantic Versioning or reliably preserve compatibility within major releases.
  - Consider this the default for mature, well-maintained dependencies.

- **Conservative: minor-version bounds**
  - `>=1.2.3,<1.3`
  - When in doubt, this is the default
  - Use for packages with uncertain versioning discipline, known minor-release incompatibilities, or where upgrades require additional testing.
  - Use for **Django**, since its feature releases can introduce breaking changes. Prefer supported long-term releases where applicable.
  - Also consider this for pre-1.0 packages, where minor releases may introduce breaking changes.

- **Exceptional: exact-version constraints**
  - `==1.2.3`
  - Use when even patch-level compatibility cannot be trusted, or when avoiding a specific regression.
      - you can do this if a package seems sketchy, e.g. a single contributor. Missing a security patch is more likely than a hijacked package, but a hijacked package has a greater magnitude of harm

- **Rare: minimum version only**
  - `>=1.2.3`
  - Consider only when
      - small surface is completely tested for correctness
      - age, maturity, and small API surface


## More notes
- Don't use esoteric syntax like `Django~=6.0.0`, it's less explicit and easier to mess up than `>6.0.0,<6.1` 
- Review release notes for framework upgrades, especially changes to supported Python, PostgreSQL, and other external dependencies
- use `pip show <package>` or `pip list | grep <package>` to see what you have installed locally
- Turn dependabot on. The better our tests are, the easier we can merge these PRs
- security is less important in our development environments, we can pin more aggressively there (and often we have to, to find versions that work in windows, macOS, and CI)
- TODO: explore a lockfile of some sort 

## first party dependencies (phac-helpers, autocomplete)
- We'll often completely pin (`==x.y.z`) our own phac-helpers and autocomplete repo. We have low trust in our own ability to do semantic versioning!
- Any security issues from first party code are unlikely to be discovered 🤷‍♂️
- We sometimes temporarily use informally released versions, e.g. `==3.2.1somerandomstring` to facilitate non-local testing without triggering any updates
- recursive dependencies installed from first party packages should follow advice above, maybe a bit more generously (accept more upgrades) so that downstream consumers can tigthen version ranges when needed
