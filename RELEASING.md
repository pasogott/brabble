# Releasing Brabble

Releases use a signed Git tag and the existing tag-triggered GoReleaser workflow.
GitHub Releases publishes the Apple Silicon archive and `checksums.txt`; the npm
manifest is a private development wrapper. There is no Homebrew tap handoff.

1. Finalize `CHANGELOG.md` for the release and update the version in
   `cmd/brabble/main.go` and `package.json`.
2. Run `make test lint build` with the binding-matched whisper.cpp installation,
   verify formatting and `goreleaser check`, and review the changes.
3. Land the release commit and wait for CI on that exact commit. Confirm the
   version tag and GitHub Release do not already exist, then create and push a
   signed tag: `git tag -s vVERSION -m vVERSION`.
4. Wait for `.github/workflows/release.yml`. The workflow builds whisper.cpp,
   signs and notarizes its bundled libraries, then signs and notarizes the executable before
   GoReleaser publishes the archive and checksums. The release notes are the
   matching changelog section, without its version heading.
5. Download the published archive, verify its checksum and quarantine behavior,
   run `brabble --version` and `brabble doctor` with an isolated config, and
   inspect the signature and minimum macOS version. The release target is
   macOS 14; PortAudio remains a Homebrew runtime dependency. Library validation
   is disabled in the hardened executable so it can load that external library.
6. Open an empty `## [Unreleased]` section and update its comparison link to the
   released tag in a separate post-release commit.

The repository needs these GitHub Actions secrets from the existing personal
release credentials: `MACOS_SIGNING_P12` (base64 Developer ID p12),
`MACOS_SIGNING_P12_PASSWORD`, `ASC_KEY_ID`, `ASC_ISSUER_ID`, and
`ASC_PRIVATE_KEY_P8`. GoReleaser signs the executable through its built-in signing
support. Runtime libraries use a temporary keychain on the isolated Actions
runner; cleanup restores the runner's search list and deletes that keychain.
Both notarization submissions wait for Apple, and the temporary API key file is
removed in an always-run step. Local release verification does not change the
user keychain search list.

For a local packaging check without release credentials, use GoReleaser snapshot
mode with `--skip=publish,notarize`. A snapshot is not release signing proof.
