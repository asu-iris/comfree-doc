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
import mujoco
import numpy as np
from comfree_warp import cf_warp as cfwarp
import streaming as mjstream

# Load model
model_path = "path/to/model.xml"
mjm = mujoco.MjSpec.from_file(model_path).compile()
mjd = mujoco.MjData(mjm)
mujoco.mj_forward(mjm, mjd)

# Configure simulation parameters
nworld = 2  # Number of parallel worlds
njmax = 5000  # Maximum number of joints
nconmax = 1000  # Maximum number of contacts

# Initialize comfree_warp engine
m = cfwarp.put_model(mjm, 
    comfree_stiffness=[50.0, 50.0, 50.0, 50.0, 50.0],
    comfree_damping=[2.0, 2.0, 2.0, 2.0, 2.0]
)
d = cfwarp.put_data(mjm, mjd, nworld=nworld, nconmax=nconmax, njmax=njmax)

# Simulation loop
for step in range(10000):
    # Set controls
    random_ctrl = 0.0 * np.random.randn(*d.ctrl.shape)
    wp.copy(d.ctrl, wp.array(random_ctrl.astype(np.float32)))
    
    # Step simulation
    cfwarp.step(m, d)
    
    # Get results back
    cfwarp.get_data_into(mjd, mjm, d)
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
STREAM_PORT = int(os.getenv("MJSTREAM_PORT", "6000"))

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

Connect to a remote stream using the local viewer:

```bash
python local_viewer.py --host <remote_host> --port <port>
```

The viewer will:
1. Connect to the streaming server
2. Receive the model definition (XML or model path)
3. Display each received state frame in real-time
4. Automatically reconnect on connection loss


### Using SSH Tunneling for Remote Access

To view a simulation on a remote server, use SSH port forwarding:

```bash
# On your local machine, create a tunnel to the remote server
ssh -L 6000:127.0.0.1:6000 user@remote-server

# In another terminal, run the local viewer
python local_viewer.py --host 127.0.0.1 --port 6000
```

This forwards local port 6000 to the remote server's port 6000, allowing secure access to the streaming server.

### Streaming Client Options

```
usage: local_viewer.py [-h] [--host HOST] [--port PORT] [--retry-delay DELAY]

options:
  --host HOST              Remote host (pair with ssh -L, default: 127.0.0.1)
  --port PORT              TCP port exposed by the remote streamer (default: 6001)
  --retry-delay DELAY      Seconds to wait before retrying after disconnect (default: 2.0)
```

**Example with custom port and retry delay:**
```bash
python local_viewer.py --host remote.server.com --port 6000 --retry-delay 5.0
```

### Local Viewer Implementation

The `local_viewer.py` is a **client-side** viewer that connects to the streaming server and displays simulation states in real-time. It uses utilities from `streaming.py`.

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

#### Client Machine: local_viewer.py

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
streamer = mjstream.StreamServer(model_path=model_path, host="0.0.0.0", port=6000)
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
- **Client Machine**: Runs `local_viewer.py` - receives packets, renders in MuJoCo viewer

### Alternative: test_viewer.py (Local Viewer)

The `test_viewer.py` script is a **standalone local viewer** that runs on a single machine with the simulation. Unlike `local_viewer.py`, it does not connect to a remote streaming server. Instead, it:
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

**Terminal 2 - View the stream locally:**
```bash
python local_viewer.py --host 127.0.0.1 --port 6000
```

### Example 2: Remote Simulation with SSH Tunnel

**Terminal 1 - SSH to remote server and run simulation:**
```bash
ssh user@remote-server
cd /path/to/comfree_dex
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
