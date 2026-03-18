# Headless Simulation and Streaming

This page describes how to run headless simulations and stream simulation states to remote viewers.

## Headless Simulation

Headless simulation allows you to run MuJoCo simulations without a local GUI viewer, which is useful for:
- Running simulations on remote servers or compute clusters
- Batch processing and benchmarking
- High-performance simulation loops
- Headless or server environments without display support

### Running Headless Simulations

To run a headless simulation, use a script like `test_headless.py`:

```python
import os
import time

import mujoco
import numpy as np
import warp as wp

import comfree_warp.mujoco_warp as mjwarp
import comfree_warp as cfwarp
import streaming as mjstream

# Engine selection: 0=MJC, 1=MJWARP, 2=COMFREE_WARP
engine = 2

# Model path (examples in benchmark/)
model_path = "benchmark/test_data/primitives.xml"

mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# Remote streaming config (set MJSTREAM_PORT<=0 to disable)
STREAM_HOST = os.getenv("MJSTREAM_HOST", "127.0.0.1")
STREAM_PORT = int(os.getenv("MJSTREAM_PORT", "7000"))
streamer = mjstream.StreamServer(model_path=model_path, host=STREAM_HOST, port=STREAM_PORT)
if streamer.enabled:
    streamer.start()

# Warp device config (e.g. "cpu" or "cuda:0")
WARP_DEVICE = os.getenv("WARP_DEVICE", "cpu")
wp.init()
wp.set_device(WARP_DEVICE)

# Configure simulation parameters
nworld = 1
njmax = 5000
nconmax = 1000

# Initialize comfree_warp engine (or mjwarp/MJC based on `engine`)
if engine == 2:
    comfree_stiffness_vec = [0.2, 0]
    comfree_damping_vec = [0.002]
    m = cfwarp.put_model(mjm, comfree_stiffness=comfree_stiffness_vec, comfree_damping=comfree_damping_vec)
    d = cfwarp.put_data(mjm, mjd, nworld=nworld, nconmax=nconmax, njmax=njmax)
    step_fn = cfwarp.step
elif engine == 1:
    m = mjwarp.put_model(mjm)
    d = mjwarp.put_data(mjm, mjd, nworld=nworld, nconmax=nconmax, njmax=njmax)
    step_fn = mjwarp.step
else:
    step_fn = None  # use mujoco.mj_step

# Simulation loop
for step in range(10000):
    # Set controls / external forces
    random_ctrl = 0.0 * np.random.randn(*d.ctrl.shape)
    wp.copy(d.ctrl, wp.array(random_ctrl.astype(np.float32)))

    # Step simulation
    if step_fn is not None:
        step_fn(m, d)
        # Copy state back into the MuJoCo data structure
        mjwarp.get_data_into(mjd, mjm, d)
    else:
        mujoco.mj_step(mjm, mjd)

    # Stream state to connected client
    if streamer.enabled:
        streamer.send_state(mjd)
```
### Configuration Options

**Engine Selection:**
- `0` - MJC (standard MuJoCo CPU engine)
- `1` - MJWARP (MuJoCo compiled to Warp)
- `2` - COMFREE_WARP (Contact-Free Warp engine)

**Simulation Parameters:**
- `nworld` - Number of parallel simulation worlds
- `njmax` - Maximum number of joints per model
- `nconmax` - Maximum number of simultaneous contacts

**Contact-Free Engine Parameters (COMFREE_WARP):**
- `comfree_stiffness` - List of stiffness values for contact dynamics
- `comfree_damping` - List of damping values for contact dynamics

## Simulation Streaming

Streaming allows you to:
- Visualize remote simulations locally
- Monitor long-running simulations on distant servers
- Reduce computational overhead on the server (no local rendering)
- Separate simulation compute from visualization

### Architecture

The streaming system uses a **server-client** architecture:
- **Server**: Runs the headless simulation and streams state updates via TCP
- **Client**: Connects to the server and renders the received state in a local viewer

### Streaming Server Setup

Enable streaming in your headless simulation script by configuring the `StreamServer`:

```python
import os
import streaming as mjstream

# Configuration from environment variables
STREAM_HOST = os.getenv("MJSTREAM_HOST", "127.0.0.1")
STREAM_PORT = int(os.getenv("MJSTREAM_PORT", "7000"))  # set <=0 to disable streaming

# Create and start streaming server
streamer = mjstream.StreamServer(
    model_path=model_path,
    host=STREAM_HOST,
    port=STREAM_PORT
)

if streamer.enabled:
    streamer.start()

# In your simulation loop, send state after each step
for step in range(num_steps):
    # ... run simulation step ...
    if streamer.enabled:
        streamer.send_state(mjd)

# Clean up
if streamer.enabled:
    streamer.stop_connection()
```

### Streaming Client Setup

To connect to a remote `StreamServer`, write a streaming client (e.g., `local_viewer.py`) that implements the following protocol:

The streaming protocol works like this:
1. Connect to the server over TCP
2. Receive the model packet (path or XML)
3. Receive state packets and apply them to a local `mujoco.MjData` using `streaming.apply_mjdata`
4. Render using `mujoco.viewer.launch_passive`

Example command (from the repo root):

```bash
python local_viewer.py --host <remote_host> --port <port>
```

### Using SSH Tunneling for Remote Access

To view a simulation on a remote server, use SSH port forwarding:

```bash
# On your local machine, create a tunnel to the remote server
ssh -L 6000:127.0.0.1:6000 user@remote-server

# In another terminal, run the streaming client (e.g. `local_viewer.py`)
python local_viewer.py --host 127.0.0.1 --port 6000
```

This forwards local port 6000 to the remote server's port 6000, allowing secure access to the streaming server.

### Example Client Options

A streaming client can expose command-line flags similar to the following:

```
usage: local_viewer.py [-h] [--host HOST] [--port PORT] [--retry-delay DELAY]

options:
  --host HOST              Remote host (pair with ssh -L, default: 127.0.0.1)
  --port PORT              TCP port exposed by the remote streamer (default: 7000)
  --retry-delay DELAY      Seconds to wait before retrying after disconnect (default: 2.0)
```

**Example with custom port and retry delay:**
```bash
python local_viewer.py --host remote.server.com --port 6000 --retry-delay 5.0
```

### Example Streaming Client Implementation

Below is an example of what a streaming client could look like. This is not shipped as a ready-made script, but it demonstrates how to use the helpers in `streaming.py` to receive model/state packets and render them in a local MuJoCo viewer.

#### Key Components from streaming.py (Client Side)

```python
def recv_packet(sock: socket.socket):
    """Receive a packet from the streaming server."""
    header = _recvall(sock, _HEADER.size)
    (length,) = _HEADER.unpack(header)
    payload = _recvall(sock, length)
    return pickle.loads(payload)

def apply_mjdata(mjd: mujoco.MjData, state) -> None:
    """Apply received state to MuJoCo data structure."""
    mjd.time = state.get("time", mjd.time)
    if "qpos" in state:
        np.copyto(mjd.qpos, state["qpos"])
    if "qvel" in state:
        np.copyto(mjd.qvel, state["qvel"])
    if "act" in state:
        np.copyto(mjd.act, state["act"])
    if "ctrl" in state:
        np.copyto(mjd.ctrl, state["ctrl"])
```

#### Example client (local machine)

The viewer runs on the **client machine** with the display. It:
1. Connects to the server via TCP socket
2. Receives the model (path or XML)
3. Receives state updates and applies them to MuJoCo
4. Renders each state in the MuJoCo viewer

```python
import socket
import mujoco
import mujoco.viewer
import streaming as mjstream

def _stream_once(sock):
    # Receive model information from server
    first_packet = mjstream.recv_packet(sock)
    kind = first_packet.get("kind")
    if kind == "model_path":
        model_path = first_packet["path"]
        print(f"Loading model from path: {model_path}")
        mjm = mujoco.MjSpec.from_file(model_path).compile()
    elif kind == "xml":
        print("Loading model from XML blob")
        mjm = mujoco.MjSpec.from_string(first_packet["xml"]).compile()
    
    mjd = mujoco.MjData(mjm)
    mujoco.mj_forward(mjm, mjd)

    # Launch viewer
    viewer = mujoco.viewer.launch_passive(mjm, mjd)
    with viewer:
        while True:
            try:
                packet = mjstream.recv_packet(sock)
            except ConnectionError as exc:
                print(f"Connection lost: {exc}")
                break
            
            kind = packet.get("kind")
            if kind == "close":
                break
            if kind != "state":
                continue
            
            # Apply received state to local MuJoCo data
            mjstream.apply_mjdata(mjd, packet["state"])
            mujoco.mj_forward(mjm, mjd)
            viewer.sync()  # Update display

def main():
    parser = argparse.ArgumentParser(description="Connect to streaming server and view simulation")
    parser.add_argument("--host", default="127.0.0.1", help="Server host")
    parser.add_argument("--port", type=int, default=6001, help="Server port")
    parser.add_argument("--retry-delay", type=float, default=2.0, help="Retry delay on disconnect")
    args = parser.parse_args()

    print(f"Connecting to {args.host}:{args.port}...")
    while True:
        try:
            with socket.create_connection((args.host, args.port)) as sock:
                print("Connected. Rendering stream...")
                _stream_once(sock)
        except KeyboardInterrupt:
            print("\nStopped.")
            break
        except OSError as exc:
            print(f"Connection failed: {exc}")
            print(f"Retrying in {args.retry_delay}s...")
            time.sleep(args.retry_delay)
```

#### Server Machine: test_headless.py Usage

The server runs on the **server machine** (usually compute cluster):

```python
import streaming as mjstream

# Server side: pack and send state
def pack_mjdata(mjd: mujoco.MjData):
    """Package MuJoCo data for transmission to client."""
    return {
        "time": float(mjd.time),
        "qpos": np.array(mjd.qpos, copy=True),
        "qvel": np.array(mjd.qvel, copy=True),
        "act": np.array(mjd.act, copy=True),
        "ctrl": np.array(mjd.ctrl, copy=True),
        "xfrc_applied": np.array(mjd.xfrc_applied, copy=True),
    }

# In your headless simulation loop
# (In `test_headless.py`, the port is controlled by MJSTREAM_PORT and defaults to 7000)
streamer = mjstream.StreamServer(model_path=model_path, host="0.0.0.0", port=7000)
streamer.start()  # Wait for client connection

for step in range(num_steps):
    # Run simulation...
    mjwarp.step(m, d)
    
    # Send state to connected client
    if streamer.connected:
        streamer.send_state(mjd)
```

**Key Distinction:**
- **Server Machine**: Runs `test_headless.py` - executes the simulation, sends state packets
- **Client Machine**: Runs a streaming client (see examples above) to receive packets and render in MuJoCo viewer

### Alternative: test_viewer.py (Local Viewer)

The `test_viewer.py` script is a **standalone local viewer** that runs on a single machine with the simulation. Unlike a dedicated streaming client, it does not connect to a remote streaming server. Instead, it:
- Loads a model locally
- Runs a simulation using the same engines (MJC, MJWARP, or COMFREE_WARP)
- Displays the simulation in real-time using MuJoCo's passive viewer

This is useful for debugging locally before deploying to remote headless simulation, or for benchmarking with visualization.



### Example 1: Local Headless Simulation with Streaming

**Terminal 1 - Run headless simulation with streaming:**
```bash
export MJSTREAM_HOST=127.0.0.1
export MJSTREAM_PORT=6000
python test_headless.py
```

**Terminal 2 - View the stream locally (using a client):**
```bash
python local_viewer.py --host 127.0.0.1 --port 6000
```

### Example 2: Remote Simulation with SSH Tunnel

**Terminal 1 - SSH to remote server and run simulation:**
```bash
ssh user@remote-server
cd /path/to/repo
export MJSTREAM_PORT=6000
python test_headless.py
```

**Terminal 2 - Create SSH tunnel (from your local machine):**
```bash
ssh -L 6000:127.0.0.1:6000 user@remote-server
```

**Terminal 3 - View the stream:**
```bash
python local_viewer.py --host 127.0.0.1 --port 6000
```

### Example 3: Benchmark Multiple Models

Run headless simulations for benchmarking without visualization overhead:

```python
import subprocess
import os

models = ["model1.xml", "model2.xml", "model3.xml"]

for model in models:
    env = os.environ.copy()
    env["MJSTREAM_PORT"] = "0"  # Disable streaming for benchmarks
    
    result = subprocess.run(
        ["python", "test_headless.py"],
        env=env,
        cwd="/path/to/simulations"
    )
```

## Troubleshooting

**Connection refused error:**
- Verify the server is running and listening on the specified port
- Check firewall settings and port exposure
- Ensure host and port match between server and client

**Model loading error:**
- Ensure model file path is correct and accessible
- If using relative paths, run from the correct directory
- Check file permissions

**Viewer closes immediately:**
- Check for Python errors in the simulation script
- Ensure model path is valid
- Review simulation parameters (nworld, njmax, nconmax)

**High latency on remote connection:**
- Check network bandwidth between server and client
- Consider reducing simulation state streaming frequency
- Use SSH tunnel compression: `ssh -C -L 6000:127.0.0.1:6000 user@remote-server`
