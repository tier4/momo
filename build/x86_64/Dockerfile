FROM ubuntu:18.04 as builder

ARG WEBRTC_VERSION

RUN apt-get update \
  && apt-get -y upgrade \
  && apt-get -y install \
    git \
    lsb-release \
    python \
    sudo \
    wget \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

RUN set -x \
  && cd /root \
  && git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git ./depot_tools

ENV PATH /root/depot_tools:$PATH
ENV DEBIAN_FRONTEND=noninteractive

RUN set -x \
  && cd /root \
  && fetch webrtc

RUN set -x \
  && apt-get update \
  && cd /root \
  && sed -i -e 's/sudo/sudo -E/g' ./src/build/install-build-deps.sh \
  && bash ./src/build/install-build-deps.sh --no-arm --no-chromeos-fonts \
  && cd src/build && git reset --hard && cd ../.. \
  && gclient sync \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

RUN set -x \
  && cd /root/src \
  && git checkout -b m$WEBRTC_VERSION remotes/branch-heads/$WEBRTC_VERSION \
  && gclient sync

RUN set -x \
  && cd /root/src \
  && gn gen out/x86_64 --args='target_os="linux" is_debug=false rtc_include_tests=false rtc_use_h264=true is_component_build=false use_rtti=true' \
  && ninja -C out/x86_64

# ビルドに必要なファイルだけ /src に移動する
# このディレクトリをマルチステージビルド先から持ってくる
RUN set -x \
  && cd /root \
  # deps
  && mkdir -p /src && cp -r src/build /src/build \
  # 生成されたライブラリ類
  && mkdir -p /src && cp -r src/out /src/out \
  # libwebrtc のヘッダ系
  && find src -maxdepth 1  -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/api          -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/base         -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/call         -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/common_audio -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/common_video -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/logging      -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/media        -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/modules      -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/p2p          -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/pc           -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/rtc_base     -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/rtc_tools    -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  && find src/video        -name '*.h' | while read line; do mkdir -p /`dirname $line`; mv $line /$line; done \
  # third_party 系
  && mkdir -p /src/third_party/llvm-build    && cp -r src/third_party/llvm-build/Release+Asserts /src/third_party/llvm-build/Release+Asserts \
  && mkdir -p /src/buildtools/third_party    && cp -r src/buildtools/third_party/libc++          /src/buildtools/third_party/libc++ \
  && mkdir -p /src/third_party               && cp -r src/third_party/libyuv                     /src/third_party/libyuv \
  && mkdir -p /src/third_party               && cp -r src/third_party/abseil-cpp                 /src/third_party/abseil-cpp \
  && mkdir -p /src/third_party/boringssl/src && cp -r src/third_party/boringssl/src/include      /src/third_party/boringssl/src/include

# Boost のビルド
COPY patch/project-config-ubuntu-x86_64.jam /root/

RUN cd ~/ \
 && set -ex \
 && curl -LO https://dl.bintray.com/boostorg/release/1.68.0/source/boost_1_68_0.tar.gz \
 && tar xf boost_1_68_0.tar.gz \
 && cd boost_1_68_0 \
 && ./bootstrap.sh \
 && cp ../project-config-ubuntu-x86_64.jam ./project-config.jam \
 && ./b2 target-os=linux architecture=arm address-model=32 link=static variant=release install -j`nproc` --prefix=/root/boost-1.68.0 --with-system --with-filesystem \
 && cd ~/ \
 && rm boost_1_68_0.tar.gz \
 && rm -rf boost_1_68_0

# 必要なファイルだけ持ってくる
FROM ubuntu:18.04

COPY --from=builder /src/ /root/src/

ENV DEBIAN_FRONTEND=noninteractive

RUN set -x \
 && apt-get update \
 && apt-get -y upgrade \
 && apt-get -y install \
     build-essential \
     git \
     lsb-release \
     sudo \
 && cd /root \
 && sed -i -e 's/sudo/sudo -E/g' ./src/build/install-build-deps.sh \
 && bash ./src/build/install-build-deps.sh --no-arm --no-chromeos-fonts \
 && cd src/build && git reset --hard && cd ../.. \
 && bash /root/src/build/install-build-deps.sh --no-arm --no-nacl --no-chromeos-fonts \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# Boost
COPY --from=builder /root/boost-1.68.0/ /root/boost-1.68.0/
