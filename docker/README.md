# Jetson Docker deployment

Target: NVIDIA Jetson AGX Orin Developer Kit, JetPack 6.2.1 / L4T 36.4.4.

Compose builds `liquid-audio-agx:local` and names the application container
`lfm`. No `.env` file is required. The host must have the NVIDIA container
runtime configured.

The Dockerfile uses NVIDIA's R36.4 JetPack image, which includes CUDA,
cuDNN, and other JetPack libraries ([NVIDIA image catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/l4t-jetpack/-/tags)).
It installs Python 3.12 and compiles PyTorch/torchaudio 2.8.0 from their
upstream release tags against JetPack's CUDA/cuDNN libraries. The Jetson CUDA
12.6 index does not supply the required PyTorch 2.8.0 Python 3.12 wheel.
Python 3.12 is required by the project, including its generic function syntax.
The build targets Orin's GPU architecture (SM 8.7) explicitly, so it does not
require GPU access during the Docker build. See the upstream
[PyTorch source build instructions](https://github.com/pytorch/pytorch/tree/v2.8.0#from-source)
and [torchaudio Jetson instructions](https://docs.pytorch.org/audio/2.8/build.jetson.html).

The first build can take several hours and needs substantial free disk space
and RAM. Compiler parallelism defaults to two jobs; set `MAX_JOBS=1` in `.env`
if compilation runs out of memory. Docker caches the GPU wheel build separately
from application changes. Build tools and source trees remain in the builder
stage, outside the final image. `BASE_IMAGE` is also an optional override.
The former `JETSON_PYPI_INDEX` override is no longer used.

Both GPU packages use the local version `2.8.0+jetson`, pinned in
`constraints.txt`, so installing demo dependencies cannot replace them with
generic PyPI builds. NCCL/MPI/TensorPipe and optional torchaudio
SoX/FFmpeg/CTC decoder extensions are disabled; the demo uses soundfile/librosa
for audio decoding and torchaudio for resampling.

Run these commands yourself **from the `docker/` directory**:

```sh
docker compose up --build -d
```

To customize the image or HTTPS hostname, copy `.env.example` to `.env`
and edit it before starting. Rebuild after changing project dependencies.

`${PWD}/..` binds the parent repository directory to `/workspace`. Python imports the
source directly from `/workspace/src`. Downloaded models and Gradio temporary
files persist under the repository's `.cache/` directory. Caddy certificates
and configuration persist in named Docker volumes.

Both services use Linux host networking so WebRTC can use the Jetson's
network interfaces directly. Compose `ports` mappings are unnecessary with
host networking. The listeners are:

| Port | Purpose |
| --- | --- |
| 80/TCP | Caddy HTTP redirects and certificate validation |
| 443/TCP | Caddy HTTPS |
| 443/UDP | Caddy HTTP/3 |
| 7860/TCP on loopback | Gradio, reached through Caddy |

Ensure ports 80 and 443 are available. For a public hostname, point its DNS
at the Jetson and forward ports 80 and 443 through any router/firewall.
For the default `https://localhost`, clients must trust Caddy's local CA;
use a real DNS hostname for remote browser access and automatic public
certificates. HTTPS permits browser microphone access.

Caddy proxies HTTP and WebSocket traffic. WebRTC audio uses separate UDP
connections; remote clients behind restrictive NAT may require additional
STUN/TURN configuration in the demo. Host networking alone does not provide
a TURN relay.

No code, builds, tests, or Docker commands were executed when preparing
these files.
