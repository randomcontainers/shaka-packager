# shaka-packager

Container images with [Shaka Packager](https://github.com/shaka-project/shaka-packager), which packages video, audio and subtitles for DASH and HLS streaming and can encrypt them on the way. `packager` writes the media files, the DASH manifest and the HLS playlists in one run. `mpd_generator` writes a DASH manifest from the media info files that `packager` can leave next to its outputs, and `pssh-box.py` creates and decodes PSSH boxes. Shaka Packager and the libraries it bundles are compiled from the release tag and its submodules. The default image also contains FFmpeg. The images are rebuilt when Shaka Packager publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by Google or the Shaka Packager project. Report problems with the image in this repository and Shaka Packager bugs at [shaka-project/shaka-packager](https://github.com/shaka-project/shaka-packager/issues).

## Quick start

Package the video and audio of `input.mp4` for DASH and HLS:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/shaka-packager \
  'in=input.mp4,stream=video,output=video.mp4,playlist_name=video.m3u8' \
  'in=input.mp4,stream=audio,output=audio.mp4,playlist_name=audio.m3u8' \
  --mpd_output manifest.mpd --hls_master_playlist_output master.m3u8
```

Each argument before the flags is a stream descriptor. `in` is the input, `stream` picks `video`, `audio`, `text` or a track number, and `output` is the file the stream is written to. With `output`, each stream is one fragmented MP4 file: the DASH manifest uses the on-demand profile, and the HLS playlists address the same files by byte range. `playlist_name` names the HLS media playlist of the stream. For one file per segment, give `init_segment` and `segment_template` instead of `output`. `--generate_static_live_mpd` keeps the manifest static for video on demand, and `--segment_duration` sets the segment length in seconds (6 by default):

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/shaka-packager \
  'in=input.mp4,stream=video,init_segment=video/init.mp4,segment_template=video/$Number$.m4s,playlist_name=video.m3u8' \
  'in=input.mp4,stream=audio,init_segment=audio/init.mp4,segment_template=audio/$Number$.m4s,playlist_name=audio.m3u8' \
  --segment_duration 4 --generate_static_live_mpd \
  --mpd_output manifest.mpd --hls_master_playlist_output master.m3u8
```

Segments start at keyframes, so encode with a keyframe interval that divides the segment duration. `packager` creates the directories in output paths. It logs to standard error. Running the image without arguments prints every flag.

A few more commands:

```sh
# Add English WebVTT subtitles from en.vtt as a text track
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/shaka-packager \
  'in=input.mp4,stream=video,output=video.mp4,playlist_name=video.m3u8' \
  'in=input.mp4,stream=audio,output=audio.mp4,playlist_name=audio.m3u8' \
  'in=en.vtt,stream=text,language=en,output=subtitles-en.vtt,playlist_name=subtitles-en.m3u8' \
  --mpd_output manifest.mpd --hls_master_playlist_output master.m3u8

# Encode 720p and 360p H.264 renditions with a keyframe every 48 frames and AAC audio
# with FFmpeg, then package them (default image)
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/shaka-packager -c '
    for h in 720 360; do
      ffmpeg -loglevel error -i input.mov -an -vf "scale=-2:$h" -c:v libx264 \
        -g 48 -keyint_min 48 -sc_threshold 0 "video-$h.mp4" || exit 1
    done
    ffmpeg -loglevel error -i input.mov -vn -c:a aac audio.mp4 &&
    packager in=video-720.mp4,stream=video,output=v720.mp4,playlist_name=v720.m3u8 \
      in=video-360.mp4,stream=video,output=v360.mp4,playlist_name=v360.m3u8 \
      in=audio.mp4,stream=audio,output=a.mp4,playlist_name=a.m3u8 \
      --mpd_output manifest.mpd --hls_master_playlist_output master.m3u8'

# Write a DASH manifest later from the media info of earlier runs
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint mpd_generator \
  ghcr.io/randomcontainers/shaka-packager \
  --input=v720.mp4.media_info,v360.mp4.media_info,a.mp4.media_info --output=manifest.mpd
```

`--output_media_info` makes `packager` write `<output>.media_info` next to each output, which is what `mpd_generator` reads. The [Shaka Packager documentation](https://shaka-project.github.io/shaka-packager/html/) describes every option, live packaging and ad insertion.

## Encryption

With keys of your own, `--enable_raw_key_encryption` encrypts each stream with the key for its DRM label, or with the key whose label is empty when there is none for it. Keys and key IDs are 32 hex digits:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/shaka-packager \
  'in=input.mp4,stream=video,output=video.mp4,playlist_name=video.m3u8' \
  'in=input.mp4,stream=audio,output=audio.mp4,playlist_name=audio.m3u8' \
  --enable_raw_key_encryption --keys 'label=:key_id=<key id>:key=<key>' \
  --protection_scheme cbcs --protection_systems Widevine,PlayReady \
  --mpd_output manifest.mpd --hls_master_playlist_output master.m3u8
```

`--protection_scheme` is `cenc` (AES-CTR) by default; FairPlay needs `cbcs`. `--protection_systems` adds the PSSH boxes of those DRM systems; without it, the files carry a Common PSSH box with the key IDs. By default the first 5 seconds, rounded up to whole segments, stay unencrypted; `--clear_lead 0` encrypts everything. Widevine and PlayReady key servers and CPIX documents are supported too, through `--enable_widevine_encryption`, `--enable_playready_encryption` and `--enable_cpix_encryption`.

Flags on the command line are visible to other users of the host in `ps` and `docker inspect`. Put them in a file instead, one per line such as `--keys=label=:key_id=<key id>:key=<key>`, and pass `--flagfile=keys.flags`.

`pssh-box.py` builds PSSH boxes for key IDs and prints the contents of existing ones:

```sh
docker run --rm --entrypoint pssh-box.py ghcr.io/randomcontainers/shaka-packager \
  --widevine-system-id --key-id 00112233445566778899aabbccddeeff --base64 > pssh.txt

docker run --rm --entrypoint pssh-box.py ghcr.io/randomcontainers/shaka-packager \
  --from-base64 "$(cat pssh.txt)" --human
```

```
PSSH Box v0
  System ID: Widevine edef8ba9-79d6-4ace-a3c8-27dcd51d21ed
  PSSH Data (size: 18):
    Widevine Data:
      Key IDs (1):
        00112233-4455-6677-8899-aabbccddeeff
```

## What is in the image

| Area | Details |
|---|---|
| Tools | `packager`, `mpd_generator` and `pssh-box.py` in `/usr/local/bin` |
| Formats | MP4 (ISO BMFF), WebM and MPEG-2 TS in and out, WebVTT and TTML subtitles, DASH manifests and HLS playlists. The [upstream README](https://github.com/shaka-project/shaka-packager#readme) lists the codecs per container |
| Encryption | The CENC schemes `cenc`, `cens`, `cbc1` and `cbcs`, and SAMPLE-AES for HLS. Keys come from the command line, a Widevine or PlayReady key server, or a CPIX document. PSSH boxes for Widevine, PlayReady and the Common system; FairPlay and Marlin are signalled without one |
| Inputs and outputs | Local files, `udp://` streams as input, and `http://` and `https://` URLs for reading and for writing with PUT |
| Libraries | protobuf, abseil, curl with Mbed TLS and c-ares, libxml2, libpng, libwebm and zlib, at the commits the release pins, linked into the programs. From the distro they load only the C and C++ runtime libraries |
| pssh-box.py | Runs on the distro's `python3` with the protobuf Python runtime of the same release, which lives in `/usr/local/lib/shaka-packager` |
| Default image | FFmpeg (`ffmpeg` and `ffprobe`), for encoding the renditions that `packager` packages |

This is a release build: `packager --version` prints `packager version v<version>-<short commit>-release`. Not included: the C++ library libpackager with its headers, the unit and integration tests, and the fully static build with mimalloc. The commit, the CMake options and the commit of every submodule are in `/usr/local/share/randomcontainers/shaka-packager/buildinfo`.

## Default or slim

Use the default image (`latest`) for general work. It adds FFmpeg, so one container can encode the renditions and package them (see [Quick start](#quick-start)). `slim` has Shaka Packager and the `python3` that `pssh-box.py` needs, without FFmpeg. Use it to build your own image, or when your media is already encoded.

The default image includes FFmpeg, which is licensed under the GNU General Public License (GPL-3.0-or-later). Use `slim` if your policy excludes GPL. The default image is also published as `ghcr.io/randomcontainers/shaka-packager-ffmpeg`, built in the [shaka-packager-ffmpeg](https://github.com/randomcontainers/shaka-packager-ffmpeg) repository with the same contents and a different digest.

## Tags

`<version>` is a Shaka Packager release such as `3.9.3`. `<minor>` and `<major>` are its shorter forms, `3.9` and `3`, and follow the newest release in that series.

| Default (with FFmpeg) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Shaka Packager version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

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

Relative paths in stream descriptors and flags are relative to `/work`. For `https://` inputs, outputs and key servers, `packager` trusts the CA certificates in `/etc/ssl/certs/ca-certificates.crt`; `--ca_file` points it to another bundle.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no FFmpeg, so FFmpeg updates do not rebuild it. The entrypoint is `["tini", "--", "packager"]`. This image runs a packaging script of your own:

```dockerfile
FROM ghcr.io/randomcontainers/shaka-packager:slim-alpine@sha256:...
COPY --chmod=0755 package-vod.sh /usr/local/bin/package-vod
ENTRYPOINT ["tini", "--", "package-vod"]
```

The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. The packages Shaka Packager needs are listed in `/usr/local/share/randomcontainers/shaka-packager/runtime-deps`. To pick up new Shaka Packager releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/shaka-packager:latest \
  --repo randomcontainers/shaka-packager --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/shaka-packager-ffmpeg` are built in that repository, so verify them with `--repo randomcontainers/shaka-packager-ffmpeg` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/shaka-packager:latest --format '{{ json .SBOM }}'
```

Shaka Packager attaches prebuilt binaries to its releases but no source archive with its submodules, and does not sign its tags, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [shaka-project/shaka-packager](https://github.com/shaka-project/shaka-packager) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. That commit records the commit of each submodule, and the build also stops unless every submodule is checked out at it. The SBOM cannot see the bundled libraries, since they are linked into the programs; their commits are in `/usr/local/share/randomcontainers/shaka-packager/buildinfo`.

## Updates

The project checks the releases of [shaka-project/shaka-packager](https://github.com/shaka-project/shaka-packager/releases) every 15 minutes. A release is picked up once it is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new FFmpeg image is published, and all of them at least every 7 days, so distro security fixes reach the current tags. The bundled libraries, curl, Mbed TLS and libxml2 among them, change only with a new Shaka Packager release, and image scanners do not report their advisories.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t shaka-packager:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. Most of the build time goes into protobuf, abseil and the other bundled libraries. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

Shaka Packager is licensed under the [BSD 3-Clause license](https://github.com/shaka-project/shaka-packager/blob/main/LICENSE), which also covers the Chromium code it includes. `packager` and `mpd_generator` contain the libraries built from its submodules as well:

- protobuf (BSD-3-Clause), with utf8_range (MIT).
- abseil-cpp (Apache-2.0).
- Mbed TLS, available under Apache-2.0 or GPL-2.0-or-later and used here under Apache-2.0.
- curl (curl license) and c-ares (MIT).
- libxml2 (MIT). Its `dict.c` and `list.c` carry an ISC-style notice instead (SPDX `ISC-Veillard`).
- zlib (Zlib), libpng (PNG Reference Library License version 2, SPDX `libpng-2.0`) and libwebm (BSD-3-Clause, with a patent grant in `PATENTS.TXT`).

`pssh-box.py` runs on protobuf's Python runtime (BSD-3-Clause). The license texts are in `/usr/local/share/randomcontainers/shaka-packager/licenses/`, and links to the source are in `/usr/local/share/randomcontainers/shaka-packager/source`. The image's license label is `BSD-3-Clause AND Apache-2.0 AND MIT AND ISC-Veillard AND curl AND Zlib AND libpng-2.0`.

`packager --licenses` prints the license files it finds in the submodules of the release, among them those of mongoose (GPL-2.0), which only the unit tests use, and mimalloc (MIT), which only fully static builds use. Neither is compiled into these images.

The default image adds FFmpeg, licensed under GPL-3.0-or-later; see [randomcontainers/ffmpeg](https://github.com/randomcontainers/ffmpeg#licenses) for its sources and license files. Python, libstdc++ and the other packages from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
