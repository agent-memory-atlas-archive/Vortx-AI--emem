# HuggingFace Space — Docker SDK wrapper around the upstream emem-server image.
# We deliberately do NOT rebuild the binary here: the multi-arch image is built
# in CI from the canonical Vortx-AI/emem repo and pushed to GHCR. This
# Dockerfile only re-exposes it on the port HF Spaces expects (default: 7860,
# overridden via app_port: 5051 in README.md frontmatter).
#
# We pin a content-addressed digest instead of `:latest` so a deletion of
# `:latest` (or an upstream regression promoted there) cannot dark the HF
# Space without warning. GHCR carries no semver tags (only latest, main and
# per-commit short-sha tags), so the digest is the only stable anchor.
# Bump it deliberately when you want the next release pulled:
#   docker buildx imagetools inspect ghcr.io/vortx-ai/emem:latest

FROM ghcr.io/vortx-ai/emem@sha256:c6e3bb00fcb76a6a43d222f78945b516feb9f519d23c40b9629a1165cfee712c

# HuggingFace runs Spaces as UID 1000 (`user`) — re-grant the runtime data
# directory and the binary's net-bind capability for that user.
USER root
RUN mkdir -p /var/emem && chown -R 1000:1000 /var/emem
USER 1000

ENV EMEM_BIND=0.0.0.0:5051 \
    EMEM_DATA=/var/emem \
    RUST_LOG=info

EXPOSE 5051

ENTRYPOINT ["/usr/local/bin/emem-server"]
