# NetDuke32 headless multiplayer host
# Pinned to NY00123's fork, tag netduke32-r11589-post_v1.2.1-345003a8d
# (same build distributed in the NukemNet "Complete Fun Pack")
#
# Key fix baked into this image: NetDuke32's timing calibration computes a
# zero per-frame increment when run under Xvfb (no real refresh rate to
# query), causing an infinite busy-wait on the very first rendered frame.
# Running a real X server (Xvnc) instead of Xvfb fixes this — Xvnc reports
# a real, configured refresh rate, which is all the calibration code needs.

# --- Stage 1: build NetDuke32 from source ---
FROM ubuntu:26.04 AS builder

RUN apt-get update && apt-get install -y \
    build-essential nasm git \
    libsdl2-dev libsdl2-mixer-dev \
    libgl1-mesa-dev libglu1-mesa-dev \
    libvpx-dev libvorbis-dev libflac-dev \
    libpng-dev libgtk2.0-dev

WORKDIR /build
RUN git clone https://voidpoint.io/NY00123/eduke32-csrefactor.git netduke32
WORKDIR /build/netduke32
RUN git checkout netduke32-r11589-post_v1.2.1-345003a8d

# This tag predates an upstream libdivide fix for stricter GCC template
# checking (GCC 15+). Without this patch the build fails with:
#   "'const class libdivide::divider<T, ALGO>' has no member named 'denom'"
RUN sed -i 's|\(return div\.denom\.magic == other\)\.denom\.magic|\1.div\.denom.magic|' \
        source/build/include/libdivide.h && \
    sed -i 's|\(div\.denom\.more == other\)\.denom\.more|\1.div\.denom.more|' \
        source/build/include/libdivide.h

RUN make netduke32 -j$(nproc)

# --- Stage 2: runtime image ---
FROM ubuntu:26.04
ARG UID=1000
ARG GID=1000

RUN apt-get update && apt-get install -y \
    tigervnc-standalone-server \
    libsdl2-2.0-0 libsdl2-mixer-2.0-0 \
    libvpx12 libvorbis0a libflac14 \
    libgl1-mesa-dri libglx-mesa0 libgl1 libglu1-mesa \
    && rm -rf /var/lib/apt/lists/*

# XDG_RUNTIME_DIR: required by the engine at startup, not created
# automatically inside a container with no login session manager.
RUN mkdir -p /tmp/runtime-${UID} \
    && chown ${UID}:${GID} /tmp/runtime-${UID} \
    && chmod 700 /tmp/runtime-${UID}

# VNC password file. Nobody actually connects a viewer to this in normal
# operation -- Xvnc just needs to exist and report a real display/refresh
# rate. A password file is still required by TigerVNC even for a
# localhost-only listener.
RUN mkdir -p /home/ubuntu/.config/tigervnc \
    && echo "netduke32" | vncpasswd -f > /home/ubuntu/.config/tigervnc/passwd \
    && chmod 600 /home/ubuntu/.config/tigervnc/passwd \
    && chown -R ${UID}:${GID} /home/ubuntu/.config/tigervnc

ENV XDG_RUNTIME_DIR=/tmp/runtime-1000
ENV SDL_AUDIODRIVER=dummy
ENV vblank_mode=0
ENV DISPLAY=:99

WORKDIR /opt/netduke32
COPY --from=builder /build/netduke32/netduke32 .
RUN chown -R ${UID}:${GID} /opt/netduke32

# Starts a real X server (Xvnc) on :99, then execs netduke32 against it.
# Using exec ensures signals (docker stop, Ctrl-C) reach the game process
# directly rather than being absorbed by this shell script.
RUN printf '#!/bin/bash\n\
Xvnc :99 -geometry 640x480 -depth 24 \\\n\
  -SecurityTypes VncAuth \\\n\
  -PasswordFile /home/ubuntu/.config/tigervnc/passwd \\\n\
  -localhost yes &\n\
sleep 1\n\
exec ./netduke32 "$@"\n' > /opt/netduke32/start.sh \
    && chmod +x /opt/netduke32/start.sh \
    && chown ${UID}:${GID} /opt/netduke32/start.sh

CMD ["/opt/netduke32/start.sh", "-nosetup", "-nologo", "-ns", "-nm"]