FROM quay.io/centos/centos:stream9 as build

# Build ganesha from source to minimize size with only VFS, install it to /usr/local
# and a use multi stage build to have a smaller image
# Set NFS_V4_RECOV_ROOT to /export

USER 0

RUN yum install -y yum-utils && \
    yum-config-manager --enable crb && \
    yum install -y https://download.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm && \
    yum install -y --setopt=tsflags=nodocs userspace-rcu-devel && \
    yum install -y --setopt=tsflags=nodocs https://rpmfind.net/linux/fedora/linux/updates/34/Everything/x86_64/Packages/l/libntirpc-3.5-1.fc34.x86_64.rpm && \
    yum install -y --setopt=tsflags=nodocs https://rpmfind.net/linux/fedora/linux/updates/34/Everything/x86_64/Packages/l/libntirpc-devel-3.5-1.fc34.x86_64.rpm && \
    INSTALL_PKGS="bison \
                    cmake \
                    dbus-devel \
                    flex \
                    tar \
                    gcc \
                    gcc-c++ \
                    git \
                    jemalloc-devel \
                    krb5-devel \
                    libacl \
                    libacl-devel \
                    libblkid-devel \
                    libnfsidmap-devel \
                    libnsl2-devel \
                    libuuid-devel \
                    patch \
                    xfsprogs-devel"  && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

# Clone specific version of ganesha
# build ganesha only supporting nfsv4 and vfs
# Set NFS_V4_RECOV_ROOT to /tmp we don't support recovery in this release
# we disable dbus (-DUSE_DBUS=OFF) for the single share manager since we don't use dynamic exports

ARG GANESHA_VERSION=V4.0.7
RUN git clone --depth 1 --branch ${GANESHA_VERSION} --recurse-submodules https://github.com/nfs-ganesha/nfs-ganesha && \
    curl -L https://github.com/nfs-ganesha/ntirpc/archive/v4.0.tar.gz | tar zx && \
    rm -rf nfs-ganesha/src/libntirpc && \
    mv ntirpc-4.0 nfs-ganesha/src/libntirpc

WORKDIR /nfs-ganesha
RUN mkdir -p /usr/local \
    && cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_CONFIG=vfs_only \
    -DUSE_DBUS=OFF -DUSE_NFS3=OFF -DUSE_NLM=OFF -DUSE_RQUOTA=OFF -DUSE_9P=OFF -D_MSPAC_SUPPORT=OFF -DRPCBIND=OFF \
    -DUSE_RADOS_RECOV=OFF -DRADOS_URLS=OFF -DUSE_FSAL_VFS=ON -DUSE_FSAL_XFS=OFF \
    -DUSE_FSAL_PROXY_V4=OFF -DUSE_FSAL_PROXY_V3=OFF -DUSE_FSAL_LUSTRE=OFF -DUSE_FSAL_LIZARDFS=OFF \
    -DUSE_FSAL_KVSFS=OFF -DUSE_FSAL_CEPH=OFF -DUSE_FSAL_GPFS=OFF -DUSE_FSAL_PANFS=OFF -DUSE_FSAL_GLUSTER=OFF \
    -DUSE_GSS=NO \
    -DCMAKE_INSTALL_PREFIX=/usr/local src/ \
	  && make \
	  && make install
RUN mkdir -p /ganesha-extra \
    && mkdir -p /ganesha-extra/etc/dbus-1/system.d \
    && cp src/scripts/ganeshactl/org.ganesha.nfsd.conf /ganesha-extra/etc/dbus-1/system.d/

FROM quay.io/centos/centos:stream9 as run

RUN yum install -y yum-utils && \
    yum-config-manager --enable crb && \
    yum install -y https://download.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm && \
    yum install -y --setopt=tsflags=nodocs userspace-rcu && \
    yum install -y --setopt=tsflags=nodocs https://rpmfind.net/linux/fedora/linux/updates/34/Everything/x86_64/Packages/l/libntirpc-3.5-1.fc34.x86_64.rpm && \
    INSTALL_PKGS="hostname \
                    jemalloc \
                    libblkid \
                    libnfsidmap \
                    libuuid \
                    nfs-utils \
                    rpcbind \
                    xfsprogs"  && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

RUN mkdir -p /var/run/dbus \
    && mkdir -p /export

# add libs from /usr/local/lib64
RUN echo /usr/local/lib64 > /etc/ld.so.conf.d/local_libs.conf

# do not ask systemd for user IDs or groups (slows down dbus-daemon start)
RUN sed -i s/systemd// /etc/nsswitch.conf

# ganesha reads /etc/mtab for mounted volumes
RUN ln -sf /proc/self/mounts /etc/mtab

COPY --from=build /usr/local /usr/local/
COPY --from=build /ganesha-extra /

ARG binary=nfs-provisioner
COPY ${binary} /nfs-provisioner

# run ldconfig after libs have been copied
RUN ldconfig

# expose mountd 20048/tcp and nfsd 2049/tcp and rpcbind 111/tcp 111/udp
EXPOSE 2049/tcp 20048/tcp 111/tcp 111/udp

ENTRYPOINT ["/nfs-provisioner"]
