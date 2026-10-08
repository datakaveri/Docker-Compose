# SPIDEr — Docker Compose Registry

Central registry of Docker Compose definitions for the applications that run
inside SPIDEr's TEE (Trusted Execution Environment).

Each application lives in its own directory with a single `docker-compose.yml`.
The middleware references the raw URL of a compose file, hands it to the TEE, and
the TEE pulls the referenced image(s) and runs the workload against the mounted
input/output volumes.

## Repository layout

```
.
├── differential-privacy/   # DP workload
│   └── docker-compose.yml
├── skald/                   # SKALD anonymisation workload
│   └── docker-compose.yml
├── skald-dicom/             # SKALD-DICOM workload
│   └── docker-compose.yml
├── _template/               # Copy this to add a new application
│   └── docker-compose.yml
├── .gitignore
├── LICENSE
└── README.md
```

## Compose conventions

Every application follows the same shape so the middleware/TEE can treat them
uniformly:

| Mount             | Container path | Purpose                    |
| ----------------- | -------------- | -------------------------- |
| `/tmp/tee_input/data`   | `/app/data`    | Input dataset (read)       |
| `/tmp/tee_input/config` | `/app/config`  | Run configuration (read)   |
| `/tmp/tee_output`       | `/app/output`  | Results written by the app |

- One directory per application, `kebab-case`, containing exactly one
  `docker-compose.yml`.
- Images should be published under `ghcr.io/datakaveri/<app>`.

## Adding a new application

1. Copy `_template/` to `<your-app>/`.
2. Fill in the service name and image in `docker-compose.yml`.
3. Keep the three standard volume mounts unless there is a specific reason not to.
4. Open a PR.

## Referenced applications

| Application          | Directory              | Image                                          | Status      |
| -------------------- | ---------------------- | ---------------------------------------------- | ----------- |
| Differential Privacy | `differential-privacy` | `ghcr.io/kailash-reddy/differential-privacy:v1`| Active      |
| SKALD                | `skald`                | `ghcr.io/datakaveri/skald:latest`              | Active      |
| SKALD-DICOM          | `skald-dicom`          | `ghcr.io/datakaveri/skald-dicom:v1.0.0`        | Active      |
| SKALD-Image          | `skald-image`          | `ghcr.io/datakaveri/skald-image:latest`        | Active      |

---

## Architecture: how the TEE consumes these files

The current proposed flow is:

```
middleware ──raw compose URL──▶ TEE ──docker pull by image ref──▶ run
```

**This flow is workable, and using this repo as the single source of truth for
compose files is the right call.** But a raw URL + mutable tag has real gaps for
a TEE, where the entire value proposition is *"you can prove exactly what code
ran on the data."* Two things undermine that today:

1. **Raw URL points at a mutable branch.** A link like
   `raw.githubusercontent.com/datakaveri/Docker-Compose/main/skald/docker-compose.yml`
   changes whenever `main` changes. What the TEE fetches tomorrow may differ from
   what was reviewed/approved today.
2. **Mutable image tags** (`:latest`, `:v1`) can be repushed to point at
   different bits. The TEE would happily run whatever the tag resolves to at
   pull time.

### Recommended hardening (in order of impact)

1. **Pin images by digest, not tag.** Use `image@sha256:<digest>` in every
   compose file. This is the single most important change — it makes the exact
   image bit-for-bit reproducible and is the foundation of TEE attestation.
2. **Reference compose files by an immutable git ref**, not a branch. Have the
   middleware use a **commit SHA** or **release tag** in the raw URL, e.g.
   `.../Docker-Compose/<commit-sha>/skald/docker-compose.yml`. Now the fetched
   file is frozen.
3. **Verify integrity before running.** The middleware should pass (or the TEE
   should compute + check) a content hash of the compose file, and ideally
   verify an image signature (e.g. [cosign](https://docs.sigstore.dev/)) so the
   TEE only runs images signed by datakaveri.
4. **Optional — distribute compose as an OCI artifact.** Instead of a raw HTTP
   URL, push the compose file to the registry alongside the image and pull both
   by digest. This unifies "what to run" and "what image" under one signed,
   content-addressed supply chain.

### Suggested target flow

```
middleware ──(compose URL @ commit-SHA  +  expected content hash)──▶ TEE
                                                                      │
                                              verify compose hash ────┤
                                              pull image by @sha256 ──┤
                                              verify cosign signature ┘
                                                                      ▼
                                                                    run
```

Start with **#1 (digest pinning)** and **#2 (immutable git ref)** — they are
cheap, require no new infrastructure, and close the biggest gaps. Signing (#3)
and OCI distribution (#4) are natural follow-ups.
