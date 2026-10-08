# Ecosystem reference

Read only the sections for ecosystems present in the project. If a command is missing, behaves differently from what is written here, or you are not sure of a flag, check the official documentation of the tool instead of guessing. Some tools below are third party and may not be installed. Check with `command -v` and fall back to the built-in alternative.

In plan mode use only the read-only entries: Discovery, Deprecations, and Dead and audit. The Upgrade entries change files and are for build mode only.

Contents: Rust, Python, JavaScript and TypeScript, Go, .NET, C and C++, GitHub Actions, Docker, pre-commit, toolchain files.

## Rust (Cargo)

- **Manifests:** `Cargo.toml` (root may have `[workspace]` and `[workspace.dependencies]`), `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]`, target-specific tables. Lockfile: `Cargo.lock`.
- **Declared floors:** `rust-version` (MSRV), `edition`.
- **Discovery:** `cargo outdated` (third-party cargo-outdated, `-R` for root dependencies only), `cargo update --dry-run`. `cargo info <crate>` shows registry metadata, but without a version it picks the one compatible with your MSRV, so it may not be the latest. For the real latest use `cargo info <crate>@<version>` or the crates.io API `/api/v1/crates/<name>`, which also gives release dates and yanked state. The `rust-version` of a release is in its `Cargo.toml` inside the local registry cache (`~/.cargo/registry/src`).
- **Upgrade:** `cargo upgrade` from third-party cargo-edit (`--dry-run` first, `--incompatible` to cross semver-breaking versions). It skips exact `=` pins unless `--pinned allow` is given. Then `cargo update` to refresh the lockfile.
- **Deprecations:** `cargo build` and `cargo clippy` warnings. `RUSTFLAGS="-D deprecated" cargo check --all-targets --all-features` turns every deprecated use into an error so none is missed.
- **Dead and audit:** `cargo audit` (RustSec, includes unmaintained and yanked), or `cargo deny check`.
- **Duplicates:** `cargo tree -d`.
- **Gotchas:** an `edition` bump is a migration of its own (`cargo fix --edition`), treat it as a major. Check each target's `rust-version` against the project's MSRV before upgrading.
- **Version format:** `serde = "1.0"` (caret by default).

## Python

- **Manifests:** `pyproject.toml` (`[project.dependencies]`, `[project.optional-dependencies]`, `[dependency-groups]`, `[tool.poetry.*]`), `requirements*.txt`, `setup.cfg`, `setup.py`, `Pipfile`. Identify the tool by lockfile: `uv.lock`, `poetry.lock`, `pdm.lock`, `Pipfile.lock`.
- **Declared floors:** `requires-python`, lower bounds in specifiers, CI matrix.
- **Discovery:** uv: `uv tree --outdated --depth 1` (shows the whole top-level tree with updates marked) or `uv pip list --outdated` (reads the existing environment, so it is only accurate if that environment is in sync, otherwise prefer `uv tree`). Poetry: `poetry show --outdated`. pip: `pip list --outdated`. Release dates and `requires_python` per release: `https://pypi.org/pypi/<name>/json`.
- **Upgrade:** uv: `uv lock --upgrade` (everything) or `uv lock --upgrade-package <pkg>` (one), then `uv sync`. These only move the lockfile within the manifest constraints, so raise a floor with `uv add "<pkg>>=X.Y"`. Poetry: `poetry update` stays within the constraints in `pyproject.toml`, while `poetry add <pkg>@latest` rewrites the constraint (as a caret on the full version, trim it). pip: `pip install -U <pkg>` then update the requirements file.
- **Deprecations:** run the test suite with `python -W error::DeprecationWarning -W error::PendingDeprecationWarning -m pytest`. Also `ruff check` with `UP` rules, and the type checker's deprecation reports (pyright `reportDeprecated`).
- **Dead and audit:** `pip-audit`. Check PyPI classifiers and the repository for "Inactive" or archived.
- **Conflicts:** `pip check` or `uv pip check`.
- **Version format:** `>=X.Y` for applications. For libraries follow the floor choice from step 1.

## JavaScript and TypeScript

- **Manifests:** `package.json` (`dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`, `overrides` or `resolutions`), `workspaces`, the `packageManager` field. Package manager by lockfile: `package-lock.json` (npm), `pnpm-lock.yaml` (pnpm), `yarn.lock` (yarn), `bun.lock` or `bun.lockb` (bun).
- **Declared floors:** `engines`, `peerDependencies`, `tsconfig` target, `.nvmrc`.
- **Discovery:** `npm outdated`, `pnpm outdated`, `bun outdated`, yarn: `yarn outdated` (v1) or `yarn upgrade-interactive`. Metadata: `npm view <pkg> time --json` (release dates), `npm view <pkg>@<ver> engines peerDependencies`, `npm view <pkg> deprecated`.
- **Upgrade:** `npm install <pkg>@latest`, `pnpm update --latest`, `bun update --latest`, then let the manager rewrite the lockfile. `npm-check-updates` is optional if installed.
- **Deprecations:** install output prints deprecated packages. For code, TypeScript alone does not report use of `@deprecated` APIs, so use ESLint with `@typescript-eslint/no-deprecated` (it needs type information, so the parser must be configured with a tsconfig), plus build and test warnings.
- **Dead and audit:** `npm audit`, `pnpm audit`. A deprecated package shows a message in `npm view <pkg> deprecated`.
- **Duplicates and peers:** `npm ls <pkg>`, `pnpm why <pkg>`.
- **Version format:** managers write `^X.Y.Z` by default. Trim to `^X.Y` in `package.json`, then resync the lockfile with the manager.

## Go

- **Manifests:** `go.mod` (`go` and `toolchain` directives, `require`, `replace`, `// indirect`), `go.work` for workspaces. `go.sum` MUST NOT be edited by hand.
- **Declared floors:** the `go` directive. In Go, versions in `go.mod` are already minimum versions.
- **Discovery:** `go list -u -m all`, `go list -m -versions <module>`, and `go list -m -u -json all` for machine-readable data: it includes the release `Time`, the available `Update`, and the `Deprecated` and `Retracted` fields. Those last two are only filled in when `-u` is passed.
- **Upgrade:** `go get <module>@latest`, `go get -u=patch ./...` for patch updates only (the flag is `-u=patch`, not `-u patch`), then `go mod tidy`. The Go version itself: `go get go@latest`, and the toolchain: `go get toolchain@patch`.
- **Deprecations:** `go vet ./...`, `staticcheck ./...` (check SA1019 for deprecated use), `golangci-lint run`. Deprecated or retracted modules show up in `go list -m -u -json all`.
- **Dead and audit:** `govulncheck ./...`.
- **Gotchas:** with the toolchain support in recent Go versions, `go get` can raise the `go` directive when a dependency requires a newer one (see go.dev/doc/toolchain). Check the `go` line after every upgrade and treat a change as a compatibility gate, not a side effect.
- **Version format:** exact `vX.Y.Z`, Go's native convention.

## .NET

- **Manifests:** `*.csproj` (`PackageReference`), `Directory.Packages.props` (central package management), `Directory.Build.props`, `packages.lock.json`, `global.json`.
- **Declared floors:** `TargetFramework`, SDK version in `global.json`.
- **Discovery:** `dotnet list package --outdated` (newer SDKs also accept the form `dotnet package list --outdated`). Add `--highest-minor` or `--highest-patch` to stay within a major, and `--include-transitive` when needed.
- **Upgrade:** `dotnet add package <name>` (latest) or with `--version`. With central package management edit the version in `Directory.Packages.props`, then `dotnet restore`.
- **Deprecations:** `dotnet list package --deprecated` for packages. For code, read the compiler warnings CS0618 and CS0612 (obsolete APIs) from `dotnet build`. To make them fail the build, set the MSBuild property: `dotnet build -p:WarningsAsErrors=CS0618`.
- **Dead and audit:** `dotnet list package --vulnerable`. It cannot be combined with `--outdated` or `--deprecated`, run it separately.
- **Version format:** NuGet treats `1.2` as a minimum of 1.2.0, so `X.Y` works natively.

## C and C++

Dependency management here is heterogeneous. Detect what the project uses first and check the tool's docs for current commands.

- **Where deps live:** `vcpkg.json` (with `builtin-baseline`), `conanfile.txt` or `conanfile.py`, CMake `FetchContent` and `find_package`, Meson wraps, git submodules, vendored sources.
- **Discovery:** vcpkg: `vcpkg x-update-baseline --dry-run` prints the planned baseline upgrade without touching files (the command is marked experimental by Microsoft, so it may change), then run it without `--dry-run` and review what changed. Conan 2: `conan graph outdated` lists the dependencies of a graph that have newer versions in a remote (it takes the same input as `conan graph info`, such as a path to the recipe). FetchContent, submodules and vendored code: compare the pinned tag or commit with upstream releases using `gh release list --repo <owner>/<repo>` or `git ls-remote --tags <url>`.
- **Deprecations:** compiler warnings, `-Wdeprecated-declarations` (on by default), `-Werror=deprecated-declarations` to catch all.
- **Gotchas:** the C or C++ standard (`CMAKE_CXX_STANDARD`) and the minimum CMake version are compatibility gates. ABI breaks may need a full rebuild.
- **Version format:** follow the tool's convention. FetchContent tags are normally exact.

## GitHub Actions

- **Where:** `.github/workflows/*.yml`, composite actions in `.github/actions/`, `uses: owner/repo@ref`.
- **Discovery:** `gh release list --repo owner/repo` or `gh api repos/owner/repo/releases/latest`.
- **Conventions:** keep the project's style. Major tags (`@v4`) stay major tags. If actions are pinned to a commit SHA, keep SHA pinning, update the SHA to the new release and update the version comment next to it.
- **Check:** release notes for runtime changes (for example the Node version the action runs on) and renamed or removed inputs. Look at recent workflow run annotations for deprecation warnings.
- **Dead:** archived action repositories. Suggest the maintained replacement if there is one.

## Docker

- **Where:** `Dockerfile*` `FROM` lines, `docker-compose*.yml` `image:` lines, CI service images.
- **Discovery:** list tags from the registry (`skopeo list-tags docker://<image>` if installed, or the registry's own API or `gh` for GitHub-hosted images).
- **Conventions:** keep the variant the project uses (`slim`, `alpine`, `bookworm`). Do not switch to `latest`. If a digest is pinned, update both the tag and the digest.
- **Check:** base image end-of-life, and the language runtime version it ships, which must stay within the project's declared floors.

## pre-commit

- **Where:** `.pre-commit-config.yaml` (`rev:` per repo).
- **Discovery:** `pre-commit autoupdate` rewrites the file, so in plan mode compare each `rev` with the upstream releases instead, using `gh release list --repo <owner>/<repo>` or `git ls-remote --tags <url>`.
- **Upgrade:** `pre-commit autoupdate` (add `--freeze` if the file already pins SHAs, and `--repo <url>` to update a single repository). Then run `pre-commit run --all-files` and report new findings, they often come from newer rules.

## Toolchain and version files

`rust-toolchain.toml`, `.tool-versions`, `.nvmrc`, `.node-version`, `.python-version`, `go` directive, `global.json`, CI matrices. Treat a runtime or language-version bump as a major upgrade: research its release notes, run the baseline checks, and make sure every dependency still supports it. Never raise a version the project's declared floor depends on without asking.