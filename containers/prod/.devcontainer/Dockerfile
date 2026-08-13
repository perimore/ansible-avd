# ============================================================
# Stage 1: Builder - compile native extensions and install deps
# ============================================================
ARG FROM_IMAGE=python
ARG FROM_VARIANT=3.12-alpine

FROM ${FROM_IMAGE}:${FROM_VARIANT} AS builder

ARG PYAVD_INSTALL_LOCATION
ARG ANSIBLE_INSTALL_LOCATION

RUN apk add --no-cache \
    gcc \
    musl-dev \
    libffi-dev \
    yaml-dev \
    cargo \
    rust \
    git \
    openssh-client \
    make

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir "pyavd[ansible] @ ${PYAVD_INSTALL_LOCATION}" \
    && pip install --no-cache-dir "msgpack>=1.2.1" "setuptools>=78.1.1"

RUN ansible-galaxy collection install ${ANSIBLE_INSTALL_LOCATION} \
    && ansible-galaxy collection install community.general

RUN pip uninstall -y pip setuptools \
    && find /opt/venv -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true \
    && find /opt/venv -type f -name "*.pyc" -delete 2>/dev/null || true \
    && find /root/.ansible -type d -name .git -exec rm -rf {} + 2>/dev/null || true \
    && rm -rf /opt/venv/share/doc /root/.cache/pip

# ============================================================
# Stage 2: Runtime - minimal production image
# ============================================================
FROM ${FROM_IMAGE}:${FROM_VARIANT}

ARG RFC3339_TIMESTAMP
ARG IMAGE_VERSION
ARG USERNAME=avd
ARG USER_UID=1000
ARG USER_GID=1000

LABEL org.opencontainers.image.description="AVD production image. Security-hardened Alpine-based container with pre-installed AVD collection."
LABEL com.arista.license_terms="https://github.com/aristanetworks/avd/blob/devel/LICENSE"
LABEL org.opencontainers.image.licenses="Apache-2.0"
LABEL org.opencontainers.image.authors="Arista AVD Team <ansible@arista.com>"
LABEL org.opencontainers.image.url="https://github.com/orgs/aristanetworks/packages?repo_name=avd"
LABEL org.opencontainers.image.documentation="https://avd.arista.com/"
LABEL org.opencontainers.image.source="https://github.com/aristanetworks/avd/tree/devel/containers"
LABEL org.opencontainers.image.created=${RFC3339_TIMESTAMP}
LABEL org.opencontainers.image.version=${IMAGE_VERSION}
LABEL org.opencontainers.image.vendor="Arista Networks"

RUN apk add --no-cache \
    git \
    openssh-client \
    make \
    yaml \
    libffi

RUN addgroup -g ${USER_GID} ${USERNAME} \
    && adduser -D -u ${USER_UID} -G ${USERNAME} -h /home/${USERNAME} -s /bin/sh ${USERNAME}

COPY --from=builder /opt/venv /opt/venv
COPY --from=builder --chown=${USER_UID}:${USER_GID} /root/.ansible/collections /home/${USERNAME}/.ansible/collections

ENV PATH="/opt/venv/bin:$PATH"
ENV ANSIBLE_CONFIG="ansible.cfg"
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Remove system Python packages shipped with the base image (they contain
# vendored copies of msgpack/setuptools with known CVEs — we use the venv only)
RUN rm -rf /usr/local/lib/python3.12/site-packages/*

COPY ./entrypoint.sh /bin/entrypoint.sh
RUN chmod +x /bin/entrypoint.sh

# Remove SUID/SGID bits from all binaries
RUN find / -perm /6000 -type f -exec chmod a-s {} + 2>/dev/null || true

USER ${USERNAME}
WORKDIR /home/${USERNAME}

ENTRYPOINT ["/bin/entrypoint.sh"]
