# Thin layer over shenxn/protonmail-bridge that adds libfido2.
#
# The Bridge updates its own binary inside the persisted state volume. Newer
# binaries link against libfido2.so.1, which the stock image does not ship, so
# after an update the container dies on start with:
#   error while loading shared libraries: libfido2.so.1 ... (exit status 127)
# Installing libfido2-1 lets the updated binary launch.
#
# The base is pinned by digest so a rebuild can't silently pull a different
# image. To bump it:
#   docker pull shenxn/protonmail-bridge:latest
#   docker image inspect shenxn/protonmail-bridge:latest --format '{{json .RepoDigests}}'
FROM shenxn/protonmail-bridge@sha256:3717b4441130675dc9131196de9f9c5287d2ea21b138d83b0486429e1737638a

RUN apt-get update \
 && apt-get install -y --no-install-recommends libfido2-1 \
 && rm -rf /var/lib/apt/lists/*
