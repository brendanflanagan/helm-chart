# IllumiDesk Grader Setup Service

## Overview

Grader setup service built as a docker image meant for deployment with IllumiDesk's Kubernetes-based
stack.

## Local Development

Images are built and tagged automatically by DockerHub. The instructions below are useful for local builds required
for testing.

### Build Docker Image

```bash
docker build --build-arg ILLUMIDESK_VERSION=1.0.0 illumidesk/grader-setup-app:latest
```

### Run the Image

Run the image locally with docker:

```bash
docker run -it --rm -p 8000:8000 illumidesk/grader-notebook-app:latest
```

Then navigate to http://localhost:8000.

### Update Dependencies

Update requirements.txt with pip-compile:

```bash
pip-compile requirements.in
```