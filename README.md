# Exaudio extension index

`index.json` is the reviewed, append-only catalog that [Exaudio](https://github.com/Nominy/exaudio)
reads by default from

```
https://raw.githubusercontent.com/Nominy/exaudio-extensions/main/index.json
```

This repository holds catalog metadata only. It is not an extension monorepo: every
extension's source, license and release assets live in its author's own GitHub
repository, and each catalog record points at one immutable `.exaudio` GitHub release
asset there. The app downloads only bytes whose SHA-256 and byte count match the record.

Currently indexed: `example.hello-panel` (the reference example from
[Nominy/exaudio-example-hello-panel](https://github.com/Nominy/exaudio-example-hello-panel)).

## Submitting an extension

Authors keep source and releases in their own repository and submit a pull request
that adds a record to `index.json`. Maintainers review it; nothing is merged
automatically, and no contributor build script or CI change is executed here.

1. Publish your source, `manifest.json` and license in a repository you own on
   GitHub. Build the WASM component with the Exaudio SDK (`sdk/rust` in the Exaudio
   repository; see `extensions/examples/hello-panel` for a complete example).
2. Package `manifest.json`, `module.wasm` and optional inert `assets/` into a ZIP
   named `<id>-<version>.exaudio`. From an Exaudio checkout:

   ```text
   cargo run -p exaudio-extension-index -- pack manifest.json COMPONENT.wasm OUTPUT.exaudio LICENSE
   ```

   No top-level directory, native binaries, executables, scripts, symlinks, absolute
   paths or `..`. Limits: 32 MiB compressed, 128 MiB expanded, 4096 entries, 256 KiB
   manifest.
3. Create a GitHub release in your repository with a path-safe tag (ASCII letters,
   digits, `.`, `_`, `-`) and upload the archive as a release asset.
4. Generate the catalog record from the exact published bytes:

   ```text
   cargo run -p exaudio-extension-index -- release-entry OWNER REPOSITORY manifest.json PACKAGE.exaudio TAG SOURCE_COMMIT
   ```

   `SOURCE_COMMIT` is the full lowercase commit ID the tag points at. The command
   validates the package and prints a complete entry; it uploads nothing.
5. Open a PR adding the printed entry to `extensions` (or appending its release to
   your existing entry). Optionally add `keywords`. Validate before pushing:

   ```text
   cargo run -p exaudio-extension-index -- validate-index index.json
   cargo run -p exaudio-extension-index -- validate-index index.json PREVIOUS_index.json
   cargo run -p exaudio-extension-index -- validate-package index.json ID VERSION PACKAGE.exaudio
   ```

   The first checks schema and release origins; the second additionally proves the
   history is append-only against the current `main` catalog; the third checks a
   downloaded asset against the record's digest, size, manifest and archive layout.
   `inspect index.json [QUERY]` prints the parsed catalog or a search result.

Maintainers verify repository ownership, license, tag-to-commit correspondence, the
exact release bytes (`validate-package` on the downloaded asset), declared
permissions and runtime compatibility. Structural validation does not replace that
review.

### Required fields

Catalog root: `schema` (currently `1`) and `extensions`.

| Extension field | Meaning |
| --- | --- |
| `id` | Dotted lowercase ID, equal to every release manifest's `id` |
| `publisher` | GitHub account, exactly equal to `repository.owner` |
| `repository` | `{ "owner", "name" }`; one repository per extension |
| `keywords` | Optional search terms |
| `releases` | Release records, one per version |

| Release field | Meaning |
| --- | --- |
| `version` | Semver, equal to `manifest.version` |
| `tag` | GitHub release tag |
| `source_commit` | Full lowercase 40- or 64-hex commit ID |
| `manifest` | The complete `manifest.json` (`runtime` must be `wasm`) |
| `asset.name` | `.exaudio` filename |
| `asset.url` | Exactly `https://github.com/OWNER/REPOSITORY/releases/download/TAG/NAME` |
| `asset.bytes` | Exact compressed byte count |
| `asset.sha256` | Lowercase 64-hex SHA-256 of the asset |
| `withdrawn` | Optional non-empty withdrawal reason |

Published records are immutable: never change the manifest, tag, commit, URL, size
or hash of an indexed version, and never delete one. Publish a new version for
changes; to withdraw a release, keep its record and set `withdrawn` to a reason. A
withdrawal blocks new installs and rollbacks to that version; it does not delete
already installed code.

## Trust boundary

Maintainers of this index review provenance; the SHA-256 binds downloaded bytes to
the reviewed record but is not a publisher signature. Only HTTPS GitHub release URLs
are fetched. The marketplace never executes repository source, installer hooks or
native code; permissions declared in a manifest are requests, granted by the user
at install time. Offline clients read their last validated cache and cannot see
later withdrawals.
