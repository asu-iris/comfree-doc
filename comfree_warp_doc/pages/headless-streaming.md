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

Connect to a remote stream using the test viewer:

```bash
python test_viewer.py --host <remote_host> --port <port>
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

# In another terminal, run the test viewer
python test_viewer.py --host 127.0.0.1 --port 6000
```

This forwards local port 6000 to the remote server's port 6000, allowing secure access to the streaming server.

### Streaming Client Options

```
usage: test_viewer.py [-h] [--host HOST] [--port PORT] [--retry-delay DELAY]

options:
  --host HOST              Remote host (default: 127.0.0.1)
  --port PORT              TCP port exposed by the remote streamer (default: 6001)
  --retry-delay DELAY      Seconds to wait before retrying after disconnect (default: 2.0)
```

**Example with custom port and retry delay:**
```bash
python test_viewer.py --host remote.server.com --port 6000 --retry-delay 5.0
```

## Workflow Examples

### Example 1: Local Headless Simulation with Streaming

**Terminal 1 - Run headless simulation with streaming:**
```bash
export MJSTREAM_HOST=127.0.0.1
export MJSTREAM_PORT=6000
python test_headless.py
```

**Terminal 2 - View the stream locally:**
```bash
python test_viewer.py --host 127.0.0.1 --port 6000
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
python test_viewer.py --host 127.0.0.1 --port 6000
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
