# Stage 1: Builder stage
FROM ubuntu AS builder

# Install necessary packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        g++ \
        golang \
        libfuse-dev \
        gcc \
        libc6-dev \
        make \
        pkg-config \
        git \
        wget \
        xz-utils \
        python3 \
        python3-pip \
        python3-venv \
        fuse

# Download rclone repository version information
ADD https://api.github.com/repos/philamp/rclone_jelly/git/refs/heads/master rclone_jelly_version.json

# Clear previous rclone build artifacts
RUN rm -rf /go/src/github.com/rclone/rclone/

# Prepare directory for building rclone
RUN mkdir -p /go/src/github.com/rclone/rclone/
WORKDIR /go/src/github.com/rclone/rclone/

# Clone rclone repository
RUN git clone https://github.com/philamp/rclone_jelly.git .

# Build rclone without CGO
RUN CGO_ENABLED=0 make

# Display rclone version
RUN ./rclone version

# Stage 2: Final stage
FROM jellyfin/jellyfin

# Docker build runtime variables (ARG) and environment variables (ENV)
ENV RAR2FS_VERSION="1.29.6"
ENV UNRAR_VERSION="6.0.7"
ENV S6_OVERLAY_VERSION="3.1.5.0"
ENV S6_OVERLAY_ARCH="x86_64"

# Copy rclone binary from builder stage
COPY --from=builder /go/src/github.com/rclone/rclone/rclone /usr/bin/rclone-linux

# Install necessary packages
RUN apt-get update && \
    apt-get -y install \
        xz-utils \
        python3 \
        python3-pip \
        python3-venv

# RAR2FS build
RUN mkdir -p /tmp/rar2fs && \
    cd /tmp/rar2fs && \
    apt-get update && \
    apt-get -y install \
        fuse \
        g++ \
        libfuse-dev \
        make \
        wget && \
    wget http://www.rarlab.com/rar/unrarsrc-${UNRAR_VERSION}.tar.gz && \
    tar zxvf unrarsrc-${UNRAR_VERSION}.tar.gz && \
    cd unrar && \
    make && \
    make install && \
    make lib && \
    make install-lib && \
    cd .. && \
    wget https://github.com/hasse69/rar2fs/releases/download/v${RAR2FS_VERSION}/rar2fs-${RAR2FS_VERSION}.tar.gz -O rar2fs-${RAR2FS_VERSION}.tar.gz && \
    tar zxvf rar2fs-${RAR2FS_VERSION}.tar.gz && \
    cd rar2fs-${RAR2FS_VERSION} && \
    ./configure --with-unrar=../unrar --with-unrar-lib=/usr/lib/ && \
    make && \
    make install && \
    rm -rf /tmp/rar2fs

# Install additional packages
RUN apt-get -y install \
        nginx \
        nginx-extras \
        unzip \
        wget

# Install s6 overlay
RUN cd /tmp && \
    wget https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz && \
    tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz && \
    wget https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_OVERLAY_ARCH}.tar.xz && \
    tar -C / -Jxpf /tmp/s6-overlay-${S6_OVERLAY_ARCH}.tar.xz

# Copy nginx configuration
COPY webdav.conf /etc/nginx/conf.d/default.conf
RUN rm /etc/nginx/sites-enabled/*

# Install ADB
RUN wget https://dl.google.com/android/repository/platform-tools-latest-linux.zip && \
    unzip platform-tools-latest-linux.zip -d /usr/local/bin && \
    rm platform-tools-latest-linux.zip

# Install build dependencies for bindfs
RUN apt-get update && \
    apt-get -y install \
        build-essential \
        pkg-config \
        autoconf \
        libtool \
        libfuse3-dev \
        libsqlite3-dev \
        cmake

# Download bindfs repository version information
ADD https://api.github.com/repos/philamp/bindfs_jelly/git/refs/heads/master bindfs_jelly_version.json

# Build bindfs
RUN mkdir -p /tmp/bindfs && \
    cd /tmp/bindfs && \
    git clone https://github.com/philamp/bindfs_jelly.git . && \
    mkdir build && \
    cd build && \
    cmake .. && \
    make clean all install && \
    rm -rf /tmp/bindfs

# Create symlink for ffprobe
RUN ln -s /usr/lib/jellyfin-ffmpeg/ffprobe /usr/local/bin/ffprobe

# Copy additional scripts
COPY root/ /root/
RUN chmod u+x /init

# Expose ports
EXPOSE 8085
EXPOSE 6502

# Define entrypoint
ENTRYPOINT ["/init"]
CMD ["python3", "/usr/local/bin/grail_jelly/main.py"]
