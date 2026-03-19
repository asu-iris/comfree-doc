# Headless Simulation and Streaming

Run MuJoCo simulations without a local GUI, and optionally stream state to a remote viewer over TCP.

## Headless Simulation

Use `test_headless.py` to run headless. Engine is selected via the `engine` variable:

```python
import os
import mujoco
import numpy as np
import warp as wp
import comfree_warp.mujoco_warp as mjwarp
import comfree_warp as cfwarp
import streaming as mjstream

engine = 2  # 0=MJC, 1=MJWARP, 2=COMFREE_WARP
model_path = "benchmark/test_data/primitives.xml"

mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# Optional streaming server (set MJSTREAM_PORT<=0 to disable)
STREAM_HOST = os.getenv("MJSTREAM_HOST", "127.0.0.1")
STREAM_PORT = int(os.getenv("MJSTREAM_PORT", "7000"))
streamer = mjstream.StreamServer(model_path=model_path, host=STREAM_HOST, port=STREAM_PORT)
if streamer.enabled:
    streamer.start()

wp.init()
wp.set_device(os.getenv("WARP_DEVICE", "cpu"))

nworld, njmax, nconmax = 1, 5000, 1000

if engine == 2:
    m = cfwarp.put_model(mjm, comfree_stiffness=[0.2, 0], comfree_damping=[0.002])
    d = cfwarp.put_data(mjm, mjd, nworld=nworld, nconmax=nconmax, njmax=njmax)
    step_fn = cfwarp.step
elif engine == 1:
    m = mjwarp.put_model(mjm)
    d = mjwarp.put_data(mjm, mjd, nworld=nworld, nconmax=nconmax, njmax=njmax)
    step_fn = mjwarp.step
else:
    step_fn = None

for step in range(10000):
    if step_fn is not None:
        step_fn(m, d)
        mjwarp.get_data_into(mjd, mjm, d)
    else:
        mujoco.mj_step(mjm, mjd)

    if streamer.enabled:
        streamer.send_state(mjd)
```

## Streaming Architecture

The streaming system is server-client over TCP:

- **Server** (`test_headless.py`): runs simulation, sends state packets
- **Client** (`local_viewer.py`): receives packets and renders in a local MuJoCo viewer

### Server Setup

```python
import streaming as mjstream

streamer = mjstream.StreamServer(model_path=model_path, host=STREAM_HOST, port=STREAM_PORT)
if streamer.enabled:
    streamer.start()

for step in range(num_steps):
    # ... simulate ...
    if streamer.enabled:
        streamer.send_state(mjd)

if streamer.enabled:
    streamer.stop_connection()
```

### Client Setup

The client connects over TCP, receives the model, then applies state packets:

```python
import socket, mujoco, mujoco.viewer
import streaming as mjstream

def _stream_once(sock):
    packet = mjstream.recv_packet(sock)
    if packet["kind"] == "model_path":
        mjm = mujoco.MjSpec.from_file(packet["path"]).compile()
    else:
        mjm = mujoco.MjSpec.from_string(packet["xml"]).compile()

    mjd = mujoco.MjData(mjm)
    mujoco.mj_forward(mjm, mjd)

    viewer = mujoco.viewer.launch_passive(mjm, mjd)
    with viewer:
        while True:
            try:
                packet = mjstream.recv_packet(sock)
            except ConnectionError:
                break
            if packet.get("kind") == "close":
                break
            if packet.get("kind") == "state":
                mjstream.apply_mjdata(mjd, packet["state"])
                mujoco.mj_forward(mjm, mjd)
                viewer.sync()
```

Run the client:
```bash
python local_viewer.py --host 127.0.0.1 --port 7000
```

Options:
```
--host HOST          Server host (default: 127.0.0.1)
--port PORT          TCP port (default: 7000)
--retry-delay DELAY  Seconds before retry on disconnect (default: 2.0)
```

### SSH Tunneling for Remote Servers

```bash
# Terminal 1 — on remote server
export MJSTREAM_PORT=6000
python test_headless.py

# Terminal 2 — on local machine, create tunnel
ssh -L 6000:127.0.0.1:6000 user@remote-server

# Terminal 3 — on local machine, view stream
python local_viewer.py --host 127.0.0.1 --port 6000
```

For better performance over slow links, enable SSH compression: `ssh -C -L ...`

## Local Viewer (no streaming)

`test_viewer.py` runs simulation and renders locally on a single machine — useful for debugging before deploying to a headless server.

## Troubleshooting

| Symptom | Check |
|---|---|
| Connection refused | Server is running; host/port match; firewall allows the port |
| Model loading error | File path is correct; run from the right directory |
| Viewer closes immediately | Check for Python errors; verify model path and simulation parameters |
| High latency | Reduce streaming frequency; use `ssh -C` compression |
