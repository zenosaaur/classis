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
  the release carries its `.sig`, and with them `package-signing-key.pub`, the
  public half of the key they verify against. Set its content as
  `[packages] verification_key`, or place the file beside the Client's program
  when you install it, and upload each archive with the signature its `.sig`
  holds; the release notes give the commands. An installation that signs with a
  key of its own keeps doing so.
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
    still refuse to share a schema. Issued certificates, signed packages,
    signing files and Gateway bindings remain valid across the upgrade.
