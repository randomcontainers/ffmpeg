# ffmpeg

Container images with `ffmpeg` and `ffprobe`, compiled from the signed [FFmpeg](https://ffmpeg.org/) release tarball against the codec libraries of Ubuntu or Alpine. The images are rebuilt when FFmpeg publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the FFmpeg project. Report problems with the image in this repository and problems with FFmpeg itself [upstream](https://ffmpeg.org/bugreports.html).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/ffmpeg -i input.mkv -c:v libx264 -crf 23 -c:a aac output.mp4
```

The same images can also be pulled as `randomcontainers.com/ffmpeg`.

Encode a video to AV1 with SVT-AV1 and Opus:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/ffmpeg -i input.mp4 -c:v libsvtav1 -preset 8 -crf 35 -c:a libopus output.webm
```

The entrypoint runs `ffmpeg` under `tini`. For `ffprobe`, override it:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --entrypoint ffprobe ghcr.io/randomcontainers/ffmpeg -hide_banner input.mp4
```

## What is in the image

| Area | Libraries |
|---|---|
| Video encoders | x264, x265, libvpx (VP8, VP9), libaom and SVT-AV1 (AV1), Theora, libwebp |
| Video decoders | dav1d (AV1), plus FFmpeg's built-in decoders |
| Audio | Opus, LAME (MP3), Vorbis, SoX resampler, and FFmpeg's own AAC encoder |
| Text and subtitles | libass, FreeType, HarfBuzz, FriBidi, fontconfig, DejaVu fonts |
| Filters | zimg (`zscale`), vid.stab, ZeroMQ (`zmq`) |
| Input and network | TLS with the system CA certificates (GnuTLS on Ubuntu, OpenSSL on Alpine), SRT, DASH (libxml2), Blu-ray |
| Hardware | VA-API and libdrm, without GPU drivers (see [Extending the slim image](#extending-the-slim-image)) |

Not included: `ffplay`, nonfree components such as fdk-aac, CUDA and NVENC, Vulkan and libplacebo, VMAF. The exact configure flags are printed by `ffmpeg -buildconf` and stored in `/usr/local/share/randomcontainers/ffmpeg/buildinfo`.

## Default or slim

FFmpeg's default image adds no other tools, so `latest` and `slim` are the same image: `ffmpeg`, `ffprobe` and the libraries they link against. Use `latest` to run it and the `slim` tags as a base for your own image. The default images of [yt-dlp](https://github.com/randomcontainers/yt-dlp), [Streamlink](https://github.com/randomcontainers/streamlink), [MediaInfo](https://github.com/randomcontainers/mediainfo), [MKVToolNix](https://github.com/randomcontainers/mkvtoolnix) and [whisper.cpp](https://github.com/randomcontainers/whisper-cpp) include this build of FFmpeg.

## Tags

`<version>` is an FFmpeg release such as `9.0.2`. `<minor>` and `<major>` are its shorter forms, `9.0` and `9`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current FFmpeg version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. The `slim`, `slim-ubuntu` and `slim-alpine` tags are rebuilt whenever FFmpeg or the distro changes. The packages FFmpeg needs are listed in `/usr/local/share/randomcontainers/ffmpeg/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/ffmpeg:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends mesa-libgallium \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

`mesa-libgallium` holds Mesa's VA-API drivers, for AMD and other GPUs that Mesa supports. For Intel GPUs on amd64, install `intel-media-va-driver`. On Alpine the packages are `mesa-va-gallium` and, on amd64 only, `intel-media-driver`. At run time, pass the GPU with `--device /dev/dri`, and because the image does not run as root, add the host group that owns `/dev/dri/renderD128` with `--group-add`. The entrypoint is `["tini", "--", "ffmpeg"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new FFmpeg releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/ffmpeg:latest \
  --repo randomcontainers/ffmpeg --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/ffmpeg:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the FFmpeg release signing key in `keys/ffmpeg-release.gpg` (fingerprint `FCF9 86EA 15E6 E293 A564 4F10 B432 2F04 D676 58D8`).

## Updates

The project checks the `n<version>` tags of [FFmpeg/FFmpeg](https://github.com/FFmpeg/FFmpeg) every 15 minutes. A release is picked up once it is 24 hours old and its tarball is on ffmpeg.org. The new version and the tarball's SHA-256 are then committed to `package.yml` and the images are rebuilt. Only the newest release is built: older series such as 8.1 and 7.1 are not, and tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t ffmpeg:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

FFmpeg in these images is built with `--enable-gpl --enable-version3`, so the binaries are licensed under the GNU General Public License, version 3 or later (GPL-3.0-or-later). They link GPL libraries (x264, x265, vid.stab) and LGPL, BSD and other libraries from the distro, and each of those packages keeps its own license. FFmpeg's license files are in `/usr/local/share/randomcontainers/ffmpeg/licenses/`.

FFmpeg's `libavcodec/jfdctfst.c`, `libavcodec/jfdctint_template.c` and `libavcodec/jrevdct.c` come from libjpeg, and the build does not change them. This software is based in part on the work of the Independent JPEG Group.

The corresponding source for each image:

- FFmpeg: every version has a GitHub release in this repository, named `v<version>`, with the exact `ffmpeg-<version>.tar.xz` that was compiled and its signature. `/usr/local/share/randomcontainers/ffmpeg/source` lists that release and the ffmpeg.org download URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The yt-dlp, Streamlink, MediaInfo and whisper.cpp default images and the combined images that include FFmpeg are built on a slim FFmpeg image from this repository. Their `com.randomcontainers.members` label records the FFmpeg version and the digest of that image, whose own `org.opencontainers.image.revision` label names the commit here.

Some formats in the image, such as H.264 and HEVC, may be covered by patents in some countries. Check what applies where you use them.

FFmpeg is a trademark of Fabrice Bellard, originator of the FFmpeg project. The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
