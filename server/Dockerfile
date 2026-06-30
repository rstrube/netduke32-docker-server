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
# This is the exact build from the NetDuke32 Complete Fun Pack (2025-04-03)
RUN git checkout netduke32-r11589-post_v1.2.1-345003a8d
RUN sed -i 's|\(return div\.denom\.magic == other\)\.denom\.magic|\1.div\.denom.magic|' source/build/include/libdivide.h && \
    sed -i 's|\(div\.denom\.more == other\)\.denom\.more|\1.div\.denom.more|' source/build/include/libdivide.h
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

RUN mkdir -p /tmp/runtime-${UID} \
    && chown ${UID}:${GID} /tmp/runtime-${UID} \
    && chmod 700 /tmp/runtime-${UID}

# Create VNC password file (arbitrary password, nobody's connecting)
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

# Startup script: launch Xvnc first, then netduke32
RUN printf '#!/bin/bash\n\
Xvnc :99 -geometry 640x480 -depth 24 \\\n\
  -SecurityTypes VncAuth \\\n\
  -PasswordFile ~/.config/tigervnc/passwd \\\n\
  -localhost yes &\n\
sleep 1\n\
exec ./netduke32 "$@"\n' > /opt/netduke32/start.sh \
    && chmod +x /opt/netduke32/start.sh \
    && chown ${UID}:${GID} /opt/netduke32/start.sh

CMD ["/opt/netduke32/start.sh", "-nosetup", "-nologo", "-ns", "-nm"]