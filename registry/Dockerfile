# Debug image for use inside a UDS/k3d cluster.
# Includes common network and inspection tools for troubleshooting other containers.
#
# Build and push to the internal Zarf registry:
#   docker build -f debug.Dockerfile -t debug-tools:latest .
#   docker save debug-tools:latest -o debug-tools.tar
#   REGISTRY=$(kubectl get secret -n zarf zarf-state -o jsonpath='{.data.state}' | base64 -d | jq -r '.registryInfo.address')
#   uds zarf tools registry push debug-tools.tar ${REGISTRY}/library/debug-tools:latest
#
# Run a one-shot debug pod (auto-deleted on exit):
#   kubectl run debug --image="${REGISTRY}/library/debug-tools:latest" --restart=Never --rm -it -- bash

FROM alpine:3.19
RUN apk add --no-cache \
    bash \
    curl \
    bind-tools \
    netcat-openbsd \
    tcpdump \
    iputils \
    busybox-extras \
    jq
