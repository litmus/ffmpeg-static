FROM public.ecr.aws/amazonlinux/amazonlinux:2023 AS build

# FROM https://trac.ffmpeg.org/wiki/CompilationGuide/Centos
RUN dnf install -y autoconf automake bzip2 bzip2-devel cmake gcc gcc-c++ git libtool make pkgconfig zlib-devel diffutils perl meson ninja-build
RUN dnf remove nasm
RUN mkdir /tmp/ffmpeg_sources
RUN mkdir /tmp/bin
ENV PATH="$PATH:/tmp/bin"
ENV PKG_CONFIG_PATH="/tmp/ffmpeg_build/lib/pkgconfig:/tmp/ffmpeg_build/lib64/pkgconfig"

## NASM
RUN cd /tmp/ffmpeg_sources \
    && curl -O -L https://www.nasm.us/pub/nasm/releasebuilds/2.15.05/nasm-2.15.05.tar.bz2 \
    && tar xjvf nasm-2.15.05.tar.bz2 \
    && cd nasm-2.15.05 \
    && ./autogen.sh \
    && ./configure --prefix="/tmp/ffmpeg_build" --bindir="/tmp/bin" \
    && make \
    && make install

## YASM
RUN cd /tmp/ffmpeg_sources \
    && curl -O -L https://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz \
    && tar xzvf yasm-1.3.0.tar.gz \
    && cd yasm-1.3.0 \
    && ./configure --prefix="/tmp/ffmpeg_build" --bindir="/tmp/bin" \
    && make \ 
    && make install

## Enviroment vars so libx264 can find dependencies
ENV NASM="/tmp/bin/nasm"
ENV YASM="/tmp/bin/yasm"
ENV AS="$NASM"

## libx264
RUN cd /tmp/ffmpeg_sources \
    && git clone --branch stable --depth 1 https://code.videolan.org/videolan/x264.git \ 
    && cd x264 \ 
    && ./configure --prefix="/tmp/ffmpeg_build" --bindir="/tmp/bin" --enable-static --disable-shared \
    && make \ 
    && make install 

## libfdk_aac
RUN cd /tmp/ffmpeg_sources \ 
    && git clone --depth 1 https://github.com/mstorsjo/fdk-aac \
    && cd fdk-aac \
    && autoreconf -fiv \
    && ./configure --prefix="/tmp/ffmpeg_build" --enable-static --disable-shared \
    && make \
    && make install

## libmp3lame
RUN cd /tmp/ffmpeg_sources \
    && curl -O -L https://downloads.sourceforge.net/project/lame/lame/3.100/lame-3.100.tar.gz \
    && tar xzvf lame-3.100.tar.gz \
    && cd lame-3.100 \
    && ./configure --prefix="/tmp/ffmpeg_build" --bindir="/tmp/bin" --enable-static --disable-shared --enable-nasm \
    && make \
    && make install

## libopus
RUN cd /tmp/ffmpeg_sources \
    && curl -O -L https://archive.mozilla.org/pub/opus/opus-1.3.1.tar.gz \
    && tar xzvf opus-1.3.1.tar.gz \
    && cd opus-1.3.1 \
    && ./configure --prefix="/tmp/ffmpeg_build" --enable-static --disable-shared \
    && make \
    && make install

## libvpx
RUN cd /tmp/ffmpeg_sources \
    && git clone --depth 1 https://chromium.googlesource.com/webm/libvpx.git \
    && cd libvpx \
    && ./configure --prefix="/tmp/ffmpeg_build" --disable-examples --disable-unit-tests --enable-vp9-highbitdepth --enable-static --disable-shared --as=yasm \
    && make \
    && make install

## av1 support
RUN cd /tmp/ffmpeg_sources \
    && git clone --branch 1.4.3 --depth 1 https://code.videolan.org/videolan/dav1d.git \
    && cd dav1d \
    && mkdir build \
    && cd build \
    && meson --default-library=static --prefix="/tmp/ffmpeg_build" --bindir="/tmp/bin" .. \
    && ninja \
    && ninja install

## ffmpeg
RUN cd /tmp/ffmpeg_sources \
    && curl -O -L https://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2 \
    && tar xjvf ffmpeg-snapshot.tar.bz2 \
    && cd ffmpeg \
    && ./configure --prefix="/tmp/ffmpeg_build" --pkg-config-flags="--static" --extra-cflags="-I/tmp/ffmpeg_build/include" --extra-ldflags="-L/tmp/ffmpeg_build/lib" --extra-libs="-lpthread -lm -lz" --bindir="/tmp/bin" --enable-gpl --enable-libfdk_aac --enable-libmp3lame --enable-libopus --enable-libvpx --enable-libx264 --enable-libdav1d --enable-nonfree --enable-yasm --enable-static --disable-shared --strip="$(type -P strip)" \
    && make \
    && make install

FROM public.ecr.aws/amazonlinux/amazonlinux:2023 AS final

COPY --from=build /tmp/bin/ffmpeg /bin/ffmpeg
COPY --from=build /tmp/bin/ffprobe /bin/ffprobe

## Run a validation test
FROM final AS test

RUN mkdir /tmp/test

# Codec: H264 - MPEG-4 AVC (part 10) (avc1)
RUN curl https://www3.cde.ca.gov/download/rod/big_buck_bunny.mp4 -o /tmp/test/h264.mp4
RUN /bin/ffmpeg -i /tmp/test/h264.mp4 -vf fps=5 -t 10 -vcodec rawvideo -pix_fmt rgb24 /tmp/test/h264_%03d.png

# Codec: MPEG-H Part2/HEVC (H.265) (hev1)
RUN curl https://www.elecard.com/storage/video/TSU_854x480.mp4 -o /tmp/test/h265.mp4
RUN /bin/ffmpeg -i /tmp/test/h264.mp4  -vf fps=5 -t 10 -vcodec rawvideo -pix_fmt rgb24 /tmp/test/h265_%03d.png

# Codec: Google/On2's VP9 Video (VP90)
RUN curl https://www.elecard.com/storage/video/UshaikaRiverEmb_854x480.webm -o /tmp/test/VP9.webm
RUN /bin/ffmpeg -i /tmp/test/VP9.webm -vf fps=5 -t 10 -vcodec rawvideo -pix_fmt rgb24 /tmp/test/VP9_%03d.png

# Codec: AOMedia's AV1 Video (av01)
RUN curl https://www.elecard.com/storage/video/CityHall_854x480.webm -o /tmp/test/AV1.webm
RUN /bin/ffmpeg -i /tmp/test/AV1.webm -vf fps=5 -t 10 -vcodec rawvideo -pix_fmt rgb24 /tmp/test/AV1_%03d.png

# publish final
FROM final AS publish
