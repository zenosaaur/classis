# Changelog

Operator-facing changes to the Server and the Client — what a running deployment
has to be told about, in particular anything that must be edited or moved before
an upgrade. The reasoning behind each change lives in the ADR it names
([`docs/adr/`](docs/adr/)); this file says only what to do.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). A
version is the one `[workspace.package] version` in `Cargo.toml` names, whose
`version/*` tag a person pushes to release it
([ADR-0012](docs/adr/0012-versions-resolved-in-the-internal-crate.md),
[ADR-0062](docs/adr/0062-the-release-run-signs-every-entry-of-the-clients-own-package-with-a-key-only-a-reviewed-version-tag-reaches.md)).
A section carries a date once its tag exists.

## [Unreleased]

### Added

- **Each release signs the Client's archives**
  ([ADR-0062](docs/adr/0062-the-release-run-signs-every-entry-of-the-clients-own-package-with-a-key-only-a-reviewed-version-tag-reaches.md)).
  Beside every `supervisor_<version>_<os>_<arch>.tar.gz`
  the release carries its `.sig`: the certificate of the release's Package
  Signer and its signature, in lowercase hex
  ([ADR-0063](docs/adr/0063-a-package-is-accepted-only-when-a-package-signer-the-compiled-in-package-root-issued-signed-it.md)).
  Upload each archive with the signature its `.sig` holds; the release notes
  give the commands. Nothing is configured for it on the Server or on any
  Client, and the release publishes no key.
- **A release starts from a pushed tag.** The `Release` workflow no longer
  creates the `version/*` tag, and a run started by hand is a dry run.

### Changed

- **The product is renamed from OpAMP Fleet to Classis**
  ([ADR-0061](docs/adr/0061-the-product-is-named-classis-and-the-strings-signatures-cover-keep-their-old-spelling.md)).
  Nothing reads an old name any more, so an installation is moved by hand
  before the upgrade.
  - **Client.** The product `opamp-fleet` is now `classis`: the package, the
    `PATH` command, the services `opamp-fleet` and `opamp-fleet-agents` (now
    `classis` and `classis-agents`), the Client's account `opamp-fleet` /
    `_opamp-fleet` / `NT SERVICE\opamp-fleet` (now `classis` / `_classis` /
    `NT SERVICE\classis`), the Managed Processes' account `opamp-fleet-agents` /
    `_opamp-fleet-agents` (now `classis-agents` / `_classis-agents`), and every
    directory named after the product — `/opt/opamp-fleet`,
    `/var/lib/opamp-fleet`, `/usr/libexec/opamp-fleet`,
    `/run/opamp-fleet-agents`, and their macOS and Windows counterparts. In this
    order:
    1. `opamp-fleet service uninstall` deregisters both old services and leaves
       the files on disk.
    2. Remove the old package (`apt remove opamp-fleet` /
       `dnf remove opamp-fleet`); on macOS and Windows, the old install root.
    3. Move the data root — `supervisor.toml`, the state directory and every
       Agent's identity — from `/var/lib/opamp-fleet` to `/var/lib/classis`, or
       the platform's counterpart.
    4. Install the `classis` package. Its post-install runs
       `service install --run-as classis`, which hands the moved files over to
       the new account; where nothing runs it, run
       `classis service install --run-as <account>` yourself. A move after the
       install is handed over by running it again.

    The old accounts `opamp-fleet` and `opamp-fleet-agents` are never removed;
    delete them once nothing runs under them.
  - **Server.** The variables `FLEET_SERVER__…` and `FLEET_SERVER_CONFIG` are
    now `CLASSIS_SERVER__…` and `CLASSIS_SERVER_CONFIG`. The image is
    `ghcr.io/neteye-platform/classis-server`, and its data directory, volume and
    default `CLASSIS_SERVER__DATA_DIR` moved from `/var/lib/opamp-fleet` to
    `/var/lib/classis`: mount the existing volume there
    (`-v <volume>:/var/lib/classis`), or the Server starts on an empty one. The
    manual's example paths are `/etc/classis`, `/etc/classis-server` and
    `/var/lib/classis-server`. Its PostgreSQL connections report the
    `application_name` `classis-server`. The UI forgets its saved style and
    theme once.
  - **Operator tool.** `opamp-fleetctl` is now `classisctl`, its release assets
    `classisctl_<version>_<os>_<arch>…`. A CA it creates from now on is named
    _Classis … CA_; existing CAs keep their names and stay valid.
  - **Telemetry.** The Client's own telemetry carries the instrumentation scope
    `classis-client`; a dashboard that filters on `opamp-fleet-client` has to
    follow.
  - **Unchanged:** the certificate URIs `urn:opamp-fleet:…`, the proof and
    signing formats `opamp-fleet-…-v1`, the Gateway's binding store, and the
    PostgreSQL advisory lock, so a Server before the rename and one after it
    still refuse to share a schema. Issued certificates and Gateway bindings
    remain valid across the upgrade. Package signatures and signing files
    survive the rename, but not the Package Roots below.
- **Package signatures are verified up to the Package Roots compiled into the
  Client and the Server**
  ([ADR-0063](docs/adr/0063-a-package-is-accepted-only-when-a-package-signer-the-compiled-in-package-root-issued-signed-it.md)).
  A signature is now the certificate of a Package Signer that a compiled-in
  Package Root issued, and that signer's Ed25519 signature over the same
  statement as before, in one DER value. The REST API
  (`PUT …/entries/{os}/{arch}/signature`, `{"signature": "<hex>"}`) and a
  release's `.sig` carry it as lowercase hex. Nothing is configured on either
  side.
  - **A signature made with a key of your own is refused**, by the Server and by
    every Client: one from `classisctl package sign`, by hand or through the
    signing file, and every 64-byte signature stored before this release.
    Today the release's Package Signer is the only one, so only the Client's
    own Package can be delivered; `otelcol`, other third-party Packages and
    builds of your own cannot, until Package Signers exist for them.
    `classisctl package keygen`, `sign` and `public-key`, the signing file,
    the package key `pki init` writes and the install's key question remain
    for now; what they make is refused or ignored.
  - **The Server refuses a signature that does not verify** with `400`, naming
    the step that failed, and stores nothing. At startup it removes every
    stored signature that no longer verifies — each one stored before this
    release, and one whose Package Signer has expired — with a warning naming
    the entry. Upload the release's `.sig` again for each entry you need.
  - **`signature_state` is `valid` or `missing`.** `invalid` and `unchecked`
    no longer occur.
  - **A Client from before this release cannot update itself to it**: it
    expects a 64-byte signature checked against its `verification_key`.
    Upgrade each installation once by hand, from the release's `.deb`, `.rpm`,
    `.msi` or archive; it updates itself from then on.
- **Every Client takes packages**
  ([ADR-0063](docs/adr/0063-a-package-is-accepted-only-when-a-package-signer-the-compiled-in-package-root-issued-signed-it.md)).
  Every Supervisor declares `AcceptsPackages`, and the Client's own Agent does
  while `[packages] self_update` consents, which is unchanged. No setting turns
  packages off: a Client without a verification key no longer takes none.

### Removed

- **`[packages] verification_key` is no longer read**, by the Client or the
  Server
  ([ADR-0063](docs/adr/0063-a-package-is-accepted-only-when-a-package-signer-the-compiled-in-package-root-issued-signed-it.md)).
  A `supervisor.toml` or `server.toml` that still holds it loads, and the
  program says once at startup that the key is ignored; remove it. The release
  no longer publishes `package-signing-key.pub`, and one beside the Client's
  program at install is still written into the file, and ignored.
