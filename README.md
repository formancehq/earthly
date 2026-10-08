# earthly

## Install Earthly

```bash
curl -sL https://get.earthly.dev | bash
```

## GoReleaser Docker v2 preset

`.goreleaser.default-v2.yaml` provides the shared release defaults with
`dockers_v2`, replacing the deprecated `dockers` and `docker_manifests` pipes.
Use GoReleaser Pro 2.18.2 and Docker Buildx with support for Linux amd64 and arm64.

To migrate a consumer, update its GoReleaser include:

```yaml
includes:
  - from_url:
      url: https://raw.githubusercontent.com/formancehq/earthly/refs/heads/main/.goreleaser.default-v2.yaml
```

Update both `build.Dockerfile` and `scratch.Dockerfile` to copy the binary from
the platform-specific context. Replace `my-binary` with the consumer's binary:

```dockerfile
ARG TARGETPLATFORM
COPY $TARGETPLATFORM/my-binary /usr/bin/my-binary
```

Also replace consumer-owned `archives.builds` with `archives.ids` and
`archives.format` with `archives.formats: [tar.gz]`, then run `goreleaser check`.

The preset preserves version and nightly tags, architecture-specific tags,
scratch variants, and `latest` for stable releases only. OCI labels, checksums,
nightly versions, and changelog rules match the original preset. Provenance and
SBOM generation remain disabled to preserve the existing container layout.

Docker v2 builds and pushes images together in the publish phase. A release with
`--skip=publish` no longer builds container images. For local image validation,
use `goreleaser release --snapshot` with a running Docker daemon; snapshot tags
receive a platform suffix and are not published.

The original `.goreleaser.default.yaml` remains available for consumers whose
Dockerfiles still expect binaries at the context root. Migrate the preset and
Dockerfiles together before removing that legacy preset.
