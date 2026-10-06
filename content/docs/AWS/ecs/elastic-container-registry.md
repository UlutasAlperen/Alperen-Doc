---
title: "elastic-container-registry"
weight: 30
---

# Elastic Container Registry

Before we can deploy any containers, we need somewhere to _store their images_. Those images need to be available all the time because containers are regularly replaced and may need to pull a fresh copy of the software.

AWS provides the [**Elastic Container Registry** (ECR)](https://aws.amazon.com/ecr/) as a managed registry service. It's like [Docker Hub](https://hub.docker.com/), but integrated directly with AWS services.

ECR helps with straightforward things like storing images in the right region, and more advanced things like controlling access with [IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html).

So let's walk through building a container image and pushing it to ECR.

## Assignment

New directive from leadership:

> **URGENT**: PatientPing is all in on ~~ai~~ containers. ChatGPT told me my infrastructure can practically run itself, and my ideas are great, and the CEO loves me.

**Build a minimal container image** that we can later use on ECR/ECS.

1. Create an `app.py` file with this content:

```python
import importlib
import os
from http.server import BaseHTTPRequestHandler, HTTPServer


def load_cmo_name() -> str | None:
    region = os.getenv("AWS_REGION") or os.getenv("AWS_DEFAULT_REGION") or "us-east-1"
    try:
        boto3 = importlib.import_module("boto3")
        ssm = boto3.client("ssm", region_name=region)
        result = ssm.get_parameter(Name="/CMO_NAME")
        value = result.get("Parameter", {}).get("Value", "").strip()
        return value or None
    except Exception:
        return None


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        cmo_name = load_cmo_name()
        if cmo_name:
            body = f"Hello from the container! From {cmo_name}."
        else:
            body = "Hello from the container!"

        payload = body.encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", str(len(payload)))
        self.end_headers()
        self.wfile.write(payload)

    def log_message(self, format, *args):
        return


if __name__ == "__main__":
    port = int(os.getenv("PORT", "8000"))
    server = HTTPServer(("0.0.0.0", port), Handler)
    server.serve_forever()
```

2. Create a `Dockerfile` for our ECS deployment, add the following content:
```dockerfile
FROM python:slim
WORKDIR /app
RUN pip install --no-cache-dir boto3
COPY app.py /app/app.py
ENTRYPOINT ["python", "/app/app.py"]
```

3. Build the image locally:

```bash
docker build --tag patientping-ecs .
```

4. Run it and ensure it works:

```bash
docker run --rm --name patientping-container -p 8000:8000 patientping-ecs
```

5. Load `http://localhost:8000/` in your browser.

    -  If `CMO_NAME` is not set, you should see `Hello from the container!`.
    -  If the task role can read `/CMO_NAME` from SSM, you should see `Hello from the container! From ${CMO_NAME}.`.

Docker builds for your local architecture by default. In the next lesson, we'll rebuild for ECS with `linux/amd64`.