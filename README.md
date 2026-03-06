# jetson-orin-nano-notes
Jetson Orin Nano setup and notes

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

One problem was that https://github.com/oobabooga/text-generation-webui installer (any of them) does NOT support aarch64, the _requirements/full/requirements.txt_ only contain for selected platforms:
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

You have to compile llama-cpp-binaries yourself for Jetson.

### nvcc
First make sure you have NVIDIA CUDA compiler installed, which was not my case due to some funny business I did.
```
nvcc -V
```
Needs to tell you a version.

### llama-cpp-binaries
Installation guide in the (README)[https://github.com/oobabooga/llama-cpp-binaries/tree/master] does not work, since running
```
git clone --recurse-submodules https://github.com/oobabooga/llama-cpp-binaries
```
will end up in a failure.
Do a regular clone instead
```
git clone https://github.com/oobabooga/llama-cpp-binaries
```
then change to the cloned directory and clone *llama.cpp* again.
```
cd llama-cpp-binaries
git clone https://github.com/ggml-org/llama.cpp.git
```

You can compile llama itself for Jetson like this (you probably don't need to)
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

The tricky part was to figure out "appropriate flag for your GPU" for llama-cpp-binaries as per https://github.com/oobabooga/llama-cpp-binaries/tree/master. The correct flag was
```
CMAKE_ARGS="-DLLAMA_CUDA=on -DCMAKE_CUDA_ARCHITECTURES=87" pip3 install -v .
```

However all attempts to compile llama binaries always only ended up in an error, breaking randomly between 20% and 40%.
I noticed the build config produced directory called _UNKNOWN.egg-info_ - that didn't look right to me.

&nbsp;

### Python3.10
I read on the internet that python3.10 might be a problem, that I may need python3.11. That was no problem, since you can happily install it with apt install.
However the only python3-pip I had was associated with python3.10, as was seen from the compilation log.

I have downloaded pip3 for python3.11 like this
```
sudo apt install python3.11-distutils
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3.11 get-pip.py
```
You can then check the pip version
```
pip3 --version
```
or 
```
python3.11 -m pip --version
```

### Clean the build directory and try again
```
cd build && make clean
cmake --build build --target clean
```
```
CMAKE_ARGS="-DLLAMA_CUDA=on -DCMAKE_CUDA_ARCHITECTURES=87" pip3 install -v .
```

I have a feeling the issue was with "venv", which was producing _UNKNOWN.egg-info_. 
I had to exit venv to correctly produce _llama_cpp_binaries.egg-info_.

&nbsp;

### Using llama-cpp-binaries in venv
If you managed to build and install _llama-cpp-binaries_, another problem is that you need to somehow make it available to venv environment in _text-generation-webui_, because you cannot very well build in venv itself.
You can manage this by enabling system-site packages to venv.
```
deactivate
rm -rf venv
python3.11 -m venv venv --system-site-packages
source venv/bin/activate
python server.py --portable --api --auto-launch
```


&nbsp;

### Pytorch
Pip seems to be installing incompatible pytorch. You need to uninstall it and install from
```
python3 -m pip install torch==2.8.0 torchvision==0.23.0 --index-url=https://pypi.jetson-ai-lab.io/jp6/cu126
```
