# Matrix Authentication Service

![Build, scan & push](https://github.com/Polarix-Containers/matrix-authentication-service/actions/workflows/build-latest.yml/badge.svg)
![Build, scan & push](https://github.com/Polarix-Containers/matrix-authentication-service/actions/workflows/build-rc.yml/badge.svg)

### Features & usage
- Drop-in replacement for the [official image](https://github.com/element-hq/matrix-authentication-service).
- Unprivileged image: you should check your volumes' permissions (eg `/data`), default UID/GID is 200014.
- We are currently not building `arm64` for this image at the moment due to it requiring really long build time. If you need an `arm64` image, please open an issue at let us know.

### Licensing
- Licensed under AGPL 3 to comply with licensing by Element.
- Any image built by Polarix Containers is provided under the combination of license terms resulting from the use of individual packages.