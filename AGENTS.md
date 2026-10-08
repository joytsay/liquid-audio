# Agent instructions

## Deployment target

- Model: NVIDIA Jetson AGX Orin Developer Kit.
- Software: JetPack 6.2.1, L4T 36.4.4.
- Run the application in a Docker container, with Caddy as the reverse proxy providing HTTPS.
- Keep code and deployment configuration compatible with this target.

## Execution restrictions

- Only change code, documentation, and configuration files.
- Do not execute application code, tests, build scripts, or Docker commands. The user will run all code and Docker commands themselves.
- Read-only file inspection is allowed when needed to prepare changes.
