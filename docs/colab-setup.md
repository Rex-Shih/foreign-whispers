# Colab Setup

Use a GPU runtime in Colab before running these cells:

```python
import sys
assert sys.version_info[:2] == (3, 11), sys.version
```

## Install Foreign Whispers

Install system packages first, then install this project from GitHub:

```python
!apt-get update -qq
!apt-get install -y -qq ffmpeg rubberband-cli imagemagick
!curl -fsSL https://deno.land/install.sh | DENO_INSTALL=/usr/local sh
%pip install -q "foreign-whispers @ git+https://github.com/Rex-Shih/foreign-whispers.git"
```

Verify the SDK import:

```python
from foreign_whispers import FWClient, BASELINE, ALIGNED

print("foreign-whispers import ok")
```

## Start Chatterbox TTS on Port 8020

The Chatterbox API defaults to port `4123`, so set `PORT=8020` explicitly:

```python
!git clone https://github.com/travisvn/chatterbox-tts-api.git /content/chatterbox-tts-api
%cd /content/chatterbox-tts-api
!curl -LsSf https://astral.sh/uv/install.sh | sh
```

```python
import os
import subprocess
import time

env = os.environ.copy()
env.update({
    "DEVICE": "cuda",
    "DEFAULT_MODEL": "multilingual",
    "PORT": "8020",
    "HOST": "0.0.0.0",
})

tts_proc = subprocess.Popen(
    ["/root/.local/bin/uv", "run", "main.py"],
    cwd="/content/chatterbox-tts-api",
    env=env,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True,
)

time.sleep(20)
print("Chatterbox TTS starting at http://127.0.0.1:8020")
```

Health check:

```python
import requests

for path in ("/health", "/status", "/docs"):
    try:
        response = requests.get(f"http://127.0.0.1:8020{path}", timeout=5)
        print(path, response.status_code)
        break
    except Exception as exc:
        print(path, exc)
```

Smoke test:

```python
import requests
from pathlib import Path
from IPython.display import Audio

response = requests.post(
    "http://127.0.0.1:8020/v1/audio/speech",
    json={"input": "Hola desde Chatterbox en Colab.", "response_format": "wav"},
    timeout=300,
)
response.raise_for_status()
Path("/content/chatterbox_test.wav").write_bytes(response.content)
Audio("/content/chatterbox_test.wav")
```

## Point Foreign Whispers at Chatterbox

If the Foreign Whispers API runs in the same Colab runtime, use the local URL:

```python
import os

os.environ["CHATTERBOX_API_URL"] = "http://127.0.0.1:8020"
os.environ["FW_TTS_BACKEND"] = "remote"
```

Then start the API on `8080`:

```python
!CHATTERBOX_API_URL=http://127.0.0.1:8020 FW_TTS_BACKEND=remote uvicorn api.src.main:app --host 0.0.0.0 --port 8080
```

In another cell or notebook process:

```python
from foreign_whispers import FWClient

fw = FWClient("http://127.0.0.1:8080")
fw.healthz()
```

## Calling Colab TTS from Your Local API

`127.0.0.1:8020` is only visible inside Colab. To use the Colab GPU TTS server
from a local Docker/API process, expose port `8020` with a tunnel, then set the
local API environment to that public HTTPS URL:

```bash
CHATTERBOX_API_URL=https://YOUR-TUNNEL-URL
FW_TTS_BACKEND=remote
```
