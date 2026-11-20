# Architecture: aura-venue-os

Version 0.1.0 · 2026-11-20 · Diagram: [ARCHITECTURE.drawio](ARCHITECTURE.drawio)

## System overview and purpose

The Venue Controller runs an embedded Linux. A platform team assembles it from a distribution: a few
hundred packages from the kernel to OpenSSL, none of them written by Aurelia Audio. This repository
holds that operating system separately from the device software, as manufacturers with a platform
team do, and in several variants.

## Components and responsibilities

| Component | Responsibility | Origin |
|---|---|---|
| Boot loader | starts the system, verifies the image if secure boot is enabled | open source, configured for the board |
| Kernel | hardware access, process isolation, network stack | open source, configured for the board |
| Base system | shell tools, C library, init system | distribution packages |
| Libraries and services | cryptography, network services, update mechanism | distribution packages |
| Variant layer | additions per variant, for example Wi-Fi support for "Controller W" | platform team |

## Key interfaces and data flows

| From | To | What |
|---|---|---|
| This repository | aura-venue-controller | an approved state of the operating system, included as linked module |
| Build system | this repository | package list, licence manifest, SPDX documents |
| Distribution security tracker | vulnerability handling | information on backported fixes |

## Infrastructure and deployment

Nothing is deployed. The repository is input for analysis.

Communication with the TrustSource API (uploads and imports from CI) uses the organisation secret
`AURA_UPLOAD_KEY`, sent as the `x-api-key` header. The base URL including the API version comes from
`TS_API_BASE_URL`; it is not hard-coded. No workflow exists yet.

Mapping to TrustSource: one project "Aura Venue OS". Its approvals are linked into the controller
project. Bills of materials can arrive in two ways, and both are exercised: a scan of the image with
`ts-scan` (native TrustSource format, upload) and the build system's SPDX or CycloneDX documents
(import).

## Architecture decision records

### ADR-001: Four repositories, operating system separate

- **Date:** 2026-11-20 (v0.1.0)
- **Status:** Accepted
- **Context:** The example has to reflect how device manufacturers organise their work: a platform
  team builds the Linux, product teams build devices on top of it, and engineering software is a
  product of its own. A single repository would hide exactly the hand-over points that cause the
  difficulties we want to study. A separate operating-system repository also leaves room for OS
  variants.
- **Decision:** Four repositories: `aura-venue-studio`, `aura-venue-controller`, `aura-venue-os`,
  `aura-venue-firmware`. In TrustSource each one is a project of its own. The operating system is
  included in the controller project as a **linked module** (a module that refers to another project
  and one of its approvals). The purchased runtime is a third-party component of the controller.
- **Consequences:** The controller depends on a specific approved state of the operating system, not
  on its source. Cross-repository topics (system overview, measurement log) need one agreed home.
  Linked-module behaviour across OS versions becomes part of what is tested (see aura-venue-os).

### ADR-002: Realistic artefacts instead of running software, measured against a ground truth

- **Date:** 2026-11-20 (v0.1.0)
- **Status:** Accepted
- **Context:** The purpose is to learn where scanning and vulnerability matching succeed and where
  manual work remains. That needs realistic input and a reference to compare with, not a working
  product.
- **Decision:** Repositories contain realistic files (manifests, vendored sources, binaries, images,
  supplier documents) that need not build or run. Each repository carries a `truth.yaml` with the
  real content and the expected vulnerabilities, each entry backed by a source. Only freely licensed
  third-party files are used; no proprietary binaries of real suppliers.
- **Consequences:** Results are reproducible and can be published. Building `truth.yaml` is real
  effort and has to be kept in step with every change to the content.

### ADR-003: Exercise both the image scan and the import with a fixed module name

- **Date:** 2026-11-20 (v0.1.0)
- **Status:** Proposed
- **Context:** For operating-system scans `ts-scan` derives the module name from the detected OS
  release. A version step therefore creates a new module in TrustSource, and settings attached to
  the old module (for example muted findings) do not carry over. For an operating system that is
  included elsewhere as a linked module and exists in several variants, this is the critical point.
  What happens to the link in that case is not known yet.
- **Decision:** Run both routes and compare them: the image scan through `ts-scan`, and the import of
  the build system's bill of materials with an explicitly fixed module name.
- **Consequences:** Twice the effort for this repository. In return the result shows which route
  keeps module identity stable across OS versions and variants, and what a manufacturer has to do
  after a base-system upgrade.

## Open decisions

- Build system and distribution: Yocto build output, or a real Debian or Alpine based image with its
  build recipe.
- Number and naming of variants and release states.
