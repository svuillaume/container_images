FROM debian:bookworm-slim

LABEL maintainer="FortiCNAPP"
LABEL description="FortiCNAPP AWS Inventory — lw_aws_inventory.sh in a container"
LABEL org.opencontainers.image.source="https://github.com/svuillaume/container_images"

# ── System dependencies ────────────────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
    bash \
    curl \
    unzip \
    jq \
    ca-certificates \
    groff \
    less \
    && rm -rf /var/lib/apt/lists/*

# ── AWS CLI v2 ─────────────────────────────────────────────────────────────────
RUN ARCH=$(uname -m) && \
    if [ "$ARCH" = "aarch64" ]; then \
      URL="https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip"; \
    else \
      URL="https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip"; \
    fi && \
    curl -fsSL "$URL" -o /tmp/awscliv2.zip && \
    unzip -q /tmp/awscliv2.zip -d /tmp && \
    /tmp/aws/install && \
    rm -rf /tmp/awscliv2.zip /tmp/aws && \
    aws --version

# ── Script ─────────────────────────────────────────────────────────────────────
WORKDIR /app
COPY lw_aws_inventory.sh .
RUN chmod +x lw_aws_inventory.sh

# ── Non-root user ──────────────────────────────────────────────────────────────
RUN useradd -m -u 1000 scanner
USER scanner

# ── Entrypoint ─────────────────────────────────────────────────────────────────
# All flags are passed directly to the script:
#   docker run --rm -e AWS_ACCESS_KEY_ID=... <image> -r us-east-1
#   docker run --rm -e AWS_ACCESS_KEY_ID=... <image> -o OrganizationAccountAccessRole
ENTRYPOINT ["/app/lw_aws_inventory.sh"]
CMD ["--help"]
