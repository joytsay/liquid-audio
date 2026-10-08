# Jetson Docker deployment

Target: NVIDIA Jetson AGX Orin Developer Kit, JetPack 6.2.1 / L4T 36.4.4.

Compose builds `liquid-audio-agx:local` and names the application container
`lfm`. No `.env` file is required. The host must have the NVIDIA container
runtime configured.

The Dockerfile uses the prebuilt community image
`dustynv/vllm:r36.4-cu129-24.04`. Its ARM64 tag was verified through the
Docker Hub registry API. Its
[published image layers](https://hub.docker.com/layers/dustynv/vllm/r36.4-cu129-24.04/images/sha256-a021e8ae522727503fefbd0d363478dcf3c2861ef54f3772b281a9108ee93427)
record Ubuntu 24.04, Python 3.12, PyTorch 2.8.0, torchaudio 2.8.0, and
Jetson Tegra CUDA 12.9.1 targeting Orin (SM 8.7). The compressed download is
about 7.2 GiB. The host remains on JetPack 6.2.1 / L4T 36.4.4; GPU operation
has not been tested here. vLLM is included in the base but the application
starts the Liquid Audio demo directly.

The previously documented `dustynv/torchaudio:2.8.0-r36.4.4-cu128-24.04`
tag is not published in Docker Hub. The available standalone torchaudio
images use 2.6.0, below this project's required version.

The application build reuses the image's interpreter and precompiled GPU
packages in its existing Python environment, including any existing venv.
It generates `/opt/constraints.txt` from the installed torch and torchaudio
metadata, preserving their exact versions and any vendor suffixes when demo
dependencies are installed. It does not compile either GPU package or download
a separate Python interpreter. Additional dependencies come from PyPI instead
of the base image's inherited Jetson index. An incompatible base image fails rather than
silently replacing its GPU packages.

BuildKit caches package downloads. Demo dependencies are installed before
application source is copied, so source edits reuse the dependency layer.
The browser UI packages are pinned in `docker/demo-constraints.txt` to the
Gradio 5.45.0, gradio-client 1.13.0, and FastRTC 0.0.32 versions in `uv.lock`,
keeping the custom WebRTC component aligned with its frontend framework.
Use normal cached builds; `--no-cache` is unnecessary.
`BASE_IMAGE` remains an optional override and must provide compatible
Python 3.12+ and GPU packages satisfying the project's requirements.
The former `MAX_JOBS` and `JETSON_PYPI_INDEX` overrides are no longer used.

Run these commands yourself **from the `docker/` directory**:

```sh
docker compose up --build -d
```

To customize the image or Hugging Face token, copy `.env.example` to `.env`
and edit it before starting. Rebuild after changing project dependencies.
If an existing `.env` sets the previous `BASE_IMAGE`, change it to
`dustynv/vllm:r36.4-cu129-24.04` or remove that override.

`${PWD}/..` binds the parent repository directory to `/workspace`. Python imports the
source directly from `/workspace/src`. Downloaded models and Gradio temporary
files persist under the repository's `.cache/` directory. Caddy certificates
and configuration persist in named Docker volumes.

The application uses Linux host networking so WebRTC can use the Jetson's
network interfaces directly. Caddy uses Docker bridge networking, with host
port 7861 mapped to container port 443. It reaches Gradio through
`host.docker.internal:7860` using Docker's host gateway. Gradio must bind to
`0.0.0.0` so the bridged proxy can reach it. The listeners are:

| Port | Purpose |
| --- | --- |
| 7861/TCP | Caddy HTTPS |
| 7861/UDP | Caddy HTTP/3 |
| 7860/TCP | Gradio, reached through Caddy |

Ensure port 7861 is available and reachable from the browser's computer.
The Caddyfile explicitly selects `192.168.5.151` for this deployment. Older
`SITE_ADDRESS` entries in `.env` are ignored; they can be removed. To change
the device address, update both the site address and `default_sni` in Caddyfile.
The supplied Caddyfile uses `tls internal` for LAN access and disables HTTP
redirects; it does not publish port 80. Public certificate issuance would
require additional configuration for certificate validation.
For the default `https://192.168.5.151:7861`, clients must trust Caddy's local CA;
use `192.168.5.151` for browser access.
HTTPS permits browser microphone access once the certificate is trusted.

The default TLS server name is `192.168.5.151`, allowing browsers that omit
SNI when connecting to an IP address to receive the correct certificate.
To export Caddy's local CA certificate, run from `docker/` on the Jetson:

```sh
docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt ./caddy-root.crt
```

Transfer `caddy-root.crt` to the computer running the browser. In Firefox,
open Settings → Privacy & Security → Certificates → View Certificates →
Authorities → Import, and trust this CA to identify websites. Reload
`https://192.168.5.151:7861` after importing it. Trust the root certificate,
not an exception for the individual page, so microphone access uses trusted
HTTPS.

After changing the Caddyfile, apply it yourself from `docker/`:

```sh
docker compose restart caddy
```

Recreate the Caddy service with `docker compose up -d --force-recreate caddy`
when changing Compose configuration, including networking, port mappings, or
environment variables. Open `https://192.168.5.151:7861` in the browser.
Caddy access logs are enabled to help diagnose route and proxy failures.

Caddy proxies HTTP and WebSocket traffic. WebRTC audio uses separate UDP
connections; remote clients behind restrictive NAT may require additional
STUN/TURN configuration in the demo. Host networking alone does not provide
a TURN relay.

No code, builds, tests, or Docker commands were executed when preparing
these files.
