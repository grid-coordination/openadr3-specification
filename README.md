# OpenADR 3 Specification

This repository contains a public copy of the [OpenADR 3 specification](https://github.com/oadr3-org/specification).

The canonical source is the [oadr3-org/specification](https://github.com/oadr3-org/specification) repository, which is currently private and requires OpenADR Alliance membership to access.  The Alliance describes its specifications as "license free but copyright of the OpenADR Alliance", and provides them at no charge, but obtaining them requires creating an account on the [OpenADR Alliance website](https://www.openadr.org/) and there is no direct link to any specification file.  See [NOTICE](NOTICE) for what each artifact's own licensing statement says.

This copy is provided so that developers, researchers, and implementers can freely access the OpenADR 3 specification artifacts — the OpenAPI YAML definitions and the Definition and User Guide documents — without a registration wall.

## Contents

Each version of the specification has its own directory:

| Version | Status | Contents |
|---------|--------|----------|
| [3.0.0](3.0.0/) | Released | OpenAPI YAML |
| [3.0.1](3.0.1/) | Released | OpenAPI YAML |
| [3.1.0](3.1.0/) | Released | OpenAPI YAML, Definition and User Guide (Markdown and PDF), enumerations |

See [VERSIONS.yaml](VERSIONS.yaml) for machine-readable version metadata.

The [doc/](doc/) directory contains additional non-normative documentation, including a comprehensive description of the additional notifications feature introduced in 3.1.0.

## Upstream provenance

Each version directory is republished from one fixed upstream commit, so a directory here always corresponds to an identifiable upstream state.

| Version | Upstream commit | Date |
|---------|-----------------|------|
| 3.0.0 | [`4907836`](https://github.com/oadr3-org/specification/commit/4907836) | 2024-11-26 |
| 3.0.1 | [`a35682b`](https://github.com/oadr3-org/specification/commit/a35682b) | 2024-11-26 |
| 3.1.0 | [`ddd2cc5`](https://github.com/oadr3-org/specification/commit/ddd2cc5) | 2025-08-07 |

Each pin is the last upstream commit to touch that version's directory.

Our 3.1.0 is taken from `ddd2cc5`, which carries document number `20250807-X` and `Document Status: Final Specification`, rather than from the upstream `v3.1.0` tag. That tag was cut on 2025-07-01, a month before the final 3.1.0 documents landed upstream, and its snapshot carries a `Definition.md` dated 09/16/2024, an earlier `User_Guide.md`, and no rendered PDFs.

Where we have applied any reformatting, the unmodified upstream bytes are preserved alongside the working copy with an `-UPSTREAM` suffix.

| File | Convention |
|------|------------|
| `<version>/openadr3.yaml` | Working copy. May include minor whitespace/indentation reformatting to improve parser interoperability. |
| `<version>/openadr3-UPSTREAM.yaml` | Byte-for-byte copy of the upstream file at the pinned commit. Present only when reformatting was applied. |

### Current reformatting

- **3.1.0**: full `yamllint` cleanup under the same config upstream applies in CI (`{extends: default, rules: {line-length: {max: 120}}}`). The upstream 3.1.0 release is frozen and won't carry this fix, so it persists in our copy. Original bytes are preserved alongside as [`3.1.0/openadr3-UPSTREAM.yaml`](3.1.0/openadr3-UPSTREAM.yaml). The `pattern:` regex and every description retain identical semantic content, apart from the two corrections called out at the end of this list. The cleanup includes:
  - Document start marker (`---`)
  - Indented `required:` list items in `BlVenRequest` and `VenVenRequest` schemas
  - Consistent 2-space indentation throughout `BlVenRequest` and `VenVenRequest` blocks (upstream used 1- and 3-space mixed indentation)
  - Comment spacing (2 spaces before `#`), bracket spacing (`[ X ]` → `[X]`), discriminator comment indentation
  - Long descriptions wrapped with a `|` block scalar
  - `pattern:` regex wrapped via double-quoted backslash line-continuation
  - **Correction 1**: the `BlVenRequest.properties.targets.default` value in upstream is `null          -` (stray trailing hyphen), which YAML parsers coerce to the **string** `"null          -"` instead of YAML `null`. The stray hyphen has been removed; this field now parses as `null` as obviously intended.
  - **Correction 2**: the `authError.properties.error.description` value in upstream begins `As described in rfc6749 |` as a plain scalar, so that `|` is a literal character in the string and the lines below it fold into a single run of text. It is now a real `|` block scalar, which drops that literal pipe and preserves the intended line structure.

## License

The specification artifacts in the versioned directories are the work of the
[OpenADR Alliance](https://www.openadr.org/), reproduced verbatim and unmodified. The OpenAPI YAML files declare the Apache License 2.0 in their own `info.license` block; the Definition and User Guide carry the Alliance's copyright notice without a separate licence grant. See [NOTICE](NOTICE).

Everything else in this repository, including the `doc/` directory and the corrections described above, is licensed under the [Apache License 2.0](LICENSE).

Additional documentation in the [doc/](doc/) directory is
Copyright (c) 2026 Clark Communications Corporation.

See [NOTICE](NOTICE) for details.
