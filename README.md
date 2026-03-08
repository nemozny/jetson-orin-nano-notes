# jetson-orin-nano-notes
Jetson Orin Nano setup and notes

## SSD Drive installation made easy
No Ubuntu laptop needed.
See - https://www.youtube.com/watch?v=497u-CcYvE8

&nbsp;

## Remote desktop
As usual, get rid of GDM3 and GNOME. Remote desktop is a huge pain with GNOME.
1. Install lightdm and lxde instead.
2. Then install tigervnc that is compatible with lightdm.
3. Configure lightdm as per https://wiki.archlinux.org/title/LightDM#VNC_Server guide. I have created a following file called "my_lightdm.conf"
```
[Seat:*]
autologin-guest=false
autologin-user=nemozny
autologin-user-timeout=0

[VNCServer]
enabled=true
command=Xvnc -rfbauth /home/nemozny/.vnc/passwd
port=5900
#listen-address=localhost
width=1920
height=1080
depth=24
```
Mind you I commented out "listen-address=localhost", because I want the VNC to listen on all interfaces.
You need to copy this file to lightdm.conf.d directory.
```
sudo cp my_lightdm.conf /etc/lightdm/lightdm.conf.d/
```
Create ~/.vnc/passwd by running
```
vncpasswd
```
Lightdm will then handle starting the VNC server.

&nbsp;

## Getting text-generation-webui to work
Loading any model based on Llama ended up in error
```
ModuleNotFoundError: No module named 'llama_cpp_binaries'
```

One problem was that https://github.com/oobabooga/text-generation-webui installer (any of them) does NOT support aarch64, the _requirements/full/requirements.txt_ only contain for wheels selected platforms:
```
# CUDA wheels
https://github.com/oobabooga/llama-cpp-binaries/releases/download/v0.74.0/llama_cpp_binaries-0.74.0+cu124-py3-none-win_amd64.whl; platform_system == "Windows"
https://github.com/oobabooga/llama-cpp-binaries/releases/download/v0.74.0/llama_cpp_binaries-0.74.0+cu124-py3-none-linux_x86_64.whl; platform_system == "Linux" and platform_machine == "x86_64"
https://github.com/turboderp-org/exllamav3/releases/download/v0.0.18/exllamav3-0.0.18+cu128.torch2.7.0-cp311-cp311-win_amd64.whl; platform_system == "Windows" and python_version == "3.11"
https://github.com/turboderp-org/exllamav3/releases/download/v0.0.18/exllamav3-0.0.18+cu128.torch2.7.0-cp311-cp311-linux_x86_64.whl; platform_system == "Linux" and platform_machine == "x86_64" and python_version == "3.11"
https://github.com/turboderp-org/exllamav2/releases/download/v0.3.2/exllamav2-0.3.2+cu128.torch2.7.0-cp311-cp311-win_amd64.whl; platform_system == "Windows" and python_version == "3.11"
https://github.com/turboderp-org/exllamav2/releases/download/v0.3.2/exllamav2-0.3.2+cu128.torch2.7.0-cp311-cp311-linux_x86_64.whl; platform_system == "Linux" and platform_machine == "x86_64" and python_version == "3.11"
https://github.com/turboderp-org/exllamav2/releases/download/v0.3.2/exllamav2-0.3.2-py3-none-any.whl; platform_system == "Linux" and platform_machine != "x86_64"
https://github.com/kingbri1/flash-attention/releases/download/v2.8.3/flash_attn-2.8.3+cu128torch2.7.0cxx11abiFALSE-cp311-cp311-win_amd64.whl; platform_system == "Windows" and python_version == "3.11"
https://github.com/Dao-AILab/flash-attention/releases/download/v2.8.3/flash_attn-2.8.3+cu12torch2.7cxx11abiFALSE-cp311-cp311-linux_x86_64.whl; platform_system == "Linux" and platform_machine == "x86_64" and python_version == "3.11"
```

You have to compile llama-cpp-binaries for Jetson yourself.

&nbsp;

### nvcc
First make sure you have NVIDIA CUDA compiler installed.
```
nvcc -V
```
Needs to tell you a version.
Otherwise look it up and install.

&nbsp;

### Download llama-cpp-binaries sources
Installation guide in https://github.com/oobabooga/llama-cpp-binaries/tree/master did not work, since running the first git clone
```
git clone --recurse-submodules https://github.com/oobabooga/llama-cpp-binaries
```
for me ended up in a failure on those "--recurse-submodules". It is caused by "llama.cpp" being a subrepository of "llama-cpp-binaries" repository.

Do a regular clone instead
```
git clone https://github.com/oobabooga/llama-cpp-binaries
```
then change to the cloned directory and clone *llama.cpp* repository again.
```
cd llama-cpp-binaries
git clone https://github.com/ggml-org/llama.cpp.git
```

#### Optionally compile llama.cpp
You can compile "llama.cpp" independently of "llama-cpp-binaries" for Jetson, but there is no benefit for our purpose.
```
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp && cmake -B build \
        -DGGML_CUDA=ON \
        -DGGML_CUDA_F16=on \
        -DLLAMA_CURL=on \
        -DGGML_CUDA_FA_ALL_QUANTS=ON  \
        -DCMAKE_CUDA_ARCHITECTURES="87"
cmake --build build --config Release --parallel 8
```

### Compile llama-cpp-binaries
The tricky part was to figure out "appropriate flag for your GPU" for llama-cpp-binaries as per https://github.com/oobabooga/llama-cpp-binaries/tree/master. The correct flag was
```
CMAKE_ARGS="-DLLAMA_CUDA=on -DCMAKE_CUDA_ARCHITECTURES=87" pip3 install -v .
```

However all attempts to compile llama binaries always only ended up in an error, breaking randomly between 20% and 40% on
```
CMAKE_BUILD_TYPE=Release
c++: fatal error: Killed signal terminated program cc1plus
compilation terminated.
Killed
Killed
...
... Error 137
... Error 137
```
Error 137 is out of memory, the process was killed.
*Do not interrupt the process! Even if there were errors 137. Wait until the process fully exited. The build can actually continue with different parts of the library and skip the ones that were killed on out of memory.

After the process exited with errors, start it again and eventually it should finish successfully.*

If the process is stubborn and keeps breaking, you can try limiting parallelization:
```
export CMAKE_BUILD_PARALLEL_LEVEL=1
export MAKEFLAGS="-j1"
CMAKE_ARGS="-DLLAMA_CUDA=on -DCMAKE_CUDA_ARCHITECTURES=87 -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc" python3.10 -m pip install -v . --no-build-isolation --no-cache-dir
```
Eventually it should finish with
```
Successfully built llama_cpp_binaries
Installing collected packages: llama_cpp_binaries
Successfully installed llama_cpp_binaries-0.87.0
```

&nbsp;

### Using llama-cpp-binaries in venv
If you managed to build and install _llama-cpp-binaries_, another problem is that you need to somehow make it available to venv environment in _text-generation-webui_, because you cannot very well build in venv itself.
You can manage this by enabling system-site packages to venv.

DO NOT USE Python 3.11 or any other Python version in venv.
Pytorch (further on) was compiled against Python 3.10 and will NOT work with Python 3.11.
```
deactivate
rm -rf venv
python3.10 -m venv venv --system-site-packages
source venv/bin/activate
pip install -r requirements/full/requirements.txt
pip install matplotlib numpy scipy --upgrade
python server.py --listen
```


&nbsp;

### Pytorch
Default Pip seems to be installing incompatible pytorch - with CPU support only and without CUDA. You need to uninstall it and install a proper version from Jetson repository.

```
pip install torch torchvision flash-attn --index-url https://pypi.jetson-ai-lab.io/jp6/cu126
```

&nbsp;

### Fix cudaMalloc out of memory issue
There was a bug introduced in Jetpack r36.4.7 that caused 
```
cudaMalloc failed: out of memory
```
errors when trying to load LLM models. For a fix see - https://forums.developer.nvidia.com/t/jetpack-6-2-2-jetson-linux-36-5-is-now-live/359622
The fix is by updating: Change etc/apt/sources.list.d/nvidia-l4t-apt-source.list from 36.4 to 36.5, then
```
sudo apt update
sudo apt dist-upgrade
```
And reboot.

&nbsp;

### ComfyUI
Works, but without some optimization, since Jetson Orin is cu126 and _"You need pytorch with cu130 or higher to use optimized CUDA operations."_
So far I found that you can use models up to 4GB, but 6GB crashed on low memory.

Notes
1. I am using "uv" instead of "python -m venv" (you need to install uv first by "pip install uv").
2. Python 3.10 because Jetson-compatible torch needs Python 3.10 (see above).
3. Uninstall incompatible torch first, that got installed through requirements.txt, and then install a proper Jetson torch.

```
git clone https://github.com/Comfy-Org/ComfyUI.git
cd ComfyUI/
uv venv --python 3.10
source .venv/bin/activate
uv pip install -r requirements.txt
uv pip uninstall torch torchvision 
uv pip install torch torchvision torchaudio --index-url https://pypi.jetson-ai-lab.io/jp6/cu126
uv run main.py --listen
```
#### Install models to ComfyUI
Taken from [Teachings/01-ComfyUISetup](https://github.com/Teachings/AIServerSetup/blob/main/05-Jetson%20Orin%20Nano%20Developer%20Kit/01-ComfyUISetup.md)

1. Navigate to [civitai.com](https://civitai.com) and select a model. For example, you can choose the following model:

   [RealVisionBabes v1.0](https://civitai.com/models/543456?modelVersionId=604282)

2. Download the model file: [realvisionbabes_v10.safetensors](https://civitai.com/api/download/models/604282?type=Model&format=SafeTensor&size=pruned&fp=fp16)

3. Place it inside the `models/checkpoints` folder.

4. Download the VAE file: [ClearVAE_V2.3_fp16.pt](https://civitai.com/api/download/models/604282?type=VAE)

5. Place it inside the `models/vae` folder.

6. Download [workflow-api.json](https://github.com/Teachings/AIServerSetup/blob/main/05-Jetson%20Orin%20Nano%20Developer%20Kit/workflow-api.json) and drag'n'drop it to ComfyUI interface.
