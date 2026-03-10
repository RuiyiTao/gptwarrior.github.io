
# Building MLC LLM & TVM for Orange Pi (Concise Guide)
https://blog.mlc.ai/2024/04/20/GPU-Accelerated-LLM-on-Orange-Pi confirms LLM models can run on Orange PI, and includes the main setups to build mlc llm and TVM. But for a beginner, there are other important setups/steps not included, but can cause failures if you don't do them right.

To be noted, as of Feb 2026, there is no pre-built package for Orange PI. Even if you run
```bash
python -m pip install --pre -U -f https://mlc.ai/wheels mlc-llm-nightly-cpu mlc-ai-nightly-cpu
```
 , you will only get an empty installation. You can follow below steps

---
## Step 1: Set up RK3588 board with OpenCL driver
Follow https://llm.mlc.ai/docs/install/gpu.html#orange-pi-5-rk3588-based-sbc.
I have set up my OrangePI to run and compile models, so all below steps are included;
- Download and install the Ubuntu 22.04 for your board from here(https://github.com/Joshua-Riek/ubuntu-rockchip/releases/tag/v1.22)
- Download and install libmali-g610.so
  ```bash
  cd /usr/lib && sudo wget https://github.com/JeffyCN/mirrors/raw/libmali/lib/aarch64-linux-gnu/libmali-valhall-g610-g6p0-x11-wayland-gbm.so
  ```
- Check if file mali_csffw.bin exist under path /lib/firmware, if not download it with command:
  ```bash
  cd /lib/firmware && sudo wget https://github.com/JeffyCN/mirrors/raw/libmali/firmware/g610/mali_csffw.bin
  ```
- Download OpenCL ICD loader and manually add libmali to ICD
  ```bash
    sudo apt update
    sudo apt install mesa-opencl-icd
    sudo mkdir -p /etc/OpenCL/vendors
    echo "/usr/lib/libmali-valhall-g610-g6p0-x11-wayland-gbm.so" | sudo tee /etc/OpenCL/vendors/mali.icd
  ```
- Download and install libOpenCL
  ```bash
  sudo apt install ocl-icd-opencl-dev
  ```
- Download and install dependencies for Mali OpenCL
  ```bash
  sudo apt install libxcb-dri2-0 libxcb-dri3-0 libwayland-client0 libwayland-server0 libx11-xcb1
  ```
- Download and install clinfo to check if OpenCL successfully installed
  ```bash
  sudo apt install clinfo
  ```
- Validate Installation
  Run `clinfo` and see if you get GPU info.
  ```bash
  $ clinfo
   arm_release_ver: g13p0-01eac0, rk_so_ver: 3
   Number of platforms                               2
   Platform Name                                   ARM Platform
   Platform Vendor                                 ARM
   Platform Version                                OpenCL 2.1 v1.g6p0-01eac0.2819f9d4dbe0b5a2f89c835d8484f9cd
   Platform Profile                                FULL_PROFILE
   ...
  ```
## Step 2: Clone and Build MLC LLM (CLI + Python Package)

It's highly recommend to create virtual environments for different purposes. Here, an environment to build mlc_llm is created.
```bash
conda create -n build-mlc-llm -c conda-forge \
    "cmake>=3.24" \
    rust \
    git \
    python=3.13

conda activate build-mlc-llm  # Activate the environment
```
### Clone repo
```bash
mkdir ~src/ && cd ~/src
git clone --recursive https://github.com/mlc-ai/mlc-llm.git
cd mlc-llm
git lfs install
mkdir -p dist/prebuilt && cd dist/prebuilt
git clone https://github.com/mlc-ai/binary-mlc-llm-libs.git lib
# download the models you want
git clone https://huggingface.co/mlc-ai/<model name>
```
### Build the `mlc_llm` Python package & CLI
```bash
cd ~/src/mlc_llm
mkdir -p build && cd build
python3 ../cmake/gen_cmake_config.py
cmake ..
cmake --build . --parallel $(nproc)
```
Expected build artifacts under ~/src/mlc_llm/build
- `libmlc_llm.so`
- `libmlc_llm_module.so`
- `mlc_llm` CLI binary

At this step, mlc_llm can't be started, since TVM has not installed yet (neither complile,nor runtime). We will test it after TVM is available.

---
## Step 3: Build TVM Runtime (Required for MLC LLM)
This build uses **TVM Unity via the Relax repo**, and don't build the one from Apache TVM. Relax Repo is forked from Apache TVM, but has a lot of features that required by MLC LLM.
This part is very important because there are specific paramters to configure, otherwise the TVM built from it can't work with mlc_llm.
### Create environment for TVM build
```bash
conda create -n build-tvm -c conda-forge \
    "llvmdev>=15" \
    "cmake>=3.24" \
    git \
    python=3.13

conda activate build-tvm # enter environment
```
### Clone TVM Unity (Relax)
```bash
cd ~/src
git clone --recursive https://github.com/mlc-ai/relax.git tvm_unity
cd tvm_unity
# optional, but good to do in order to have matched version of TVM & mlc_llm
# git checkout {commit Id of mlc_llm relax submodule}  # commit id can be retrieved from https://github.com/mlc-ai/mlc-llm/tree/main/3rdparty

```

### Configure TVM Build
```bash
mkdir -p build && cd build
cp ../cmake/config.cmake .
echo "set(CMAKE_BUILD_TYPE RelWithDebInfo)" >> config.cmake
echo "set(USE_LLVM \"llvm-config --ignore-libllvm --link-static\")" >> config.cmake
echo "set(HIDE_PRIVATE_SYMBOLS ON)" >> config.cmake
echo "set(USE_OPENCL ON)" >> config.cmake
```

### Build TVM
```bash
cmake .. && make -j $(nproc) && cd ..  # Thi will build not just runtime, but also compiler
# Alternatively, if you just want runtime, you can build with below command:
# cmake .. && cmake --build . --target runtime --parallel $(nproc) && cd ../..
```

This produces:
- `libtvm_runtime.so`
- `libtvm.so`
- `xxxx.so`


## Step 4: Run model on OrangePI
### Create environment to run mlc_llm
```bash
conda create -n mlc_llm python=3.13
conda activate mlc_llm
```
### Install MLC_LLM and TVM from your local build
Install python package from your local build has 2 methods. Here I installed the method with installation, not by setting `PYTHONPATH`
- Install mlc_llm
  ```bash
  cd ~/src/mlc_llm/python
  pip install .
  ```
  Check if `libxxx.so` files are copied to ~/conda3/lib/python3.13/site-packages/mlc_llm/ folder. If not, copy those fiels from ~/src/mlc_llm/build to ~/conda3/lib/python3.13/site-packages/mlc_llm/

  Confirm installation:
    ```bash
    python3 - << 'EOF'
    import mlc_llm
    print(mlc_llm.__file__)
    EOF
    ```
    It should point to the **** path.

  **Note**, No dependencies of the package will be installed in this process. You are very likely to run into errors related with missing dependency. You can manually install them per error message.
  Also important, `flashinfer-python` is for CUDA, you must not install it. Otherwise, you may run into issue.

- Install TVM
  ```bash
  cd ~/src/tvm_unity/python
  pip install .
  ```
Check if `libxxx.so` files are copied to ~/conda3/lib/python3.13/site-packages/tvm/ folder. If not, copy those fiels from ~/src/mlc_llm/build to ~/conda3/lib/python3.13/site-packages/tvm/

Confirm installation:
```bash
python3 - << 'EOF'
import tvm
print(tvm.__file__)
EOF
```
It should point to the **** path.

- Install tvm_ffi
  This is also a required package. To avoid version conflict issues, you should install it from ~/src/tvm_unity/tvm_ffi folder, and run `pip install .`

---
## Step 5. Ready to Use MLC LLM

```
conda activate mlc_llm
cd ~/src/mlc_llm/dist/prebuilt  # the mmodels are downloaded here
python3 - << 'EOF'
    from mlc_llm import MLCEngine
    from mlc_llm.serve.config import EngineConfig
    model = "<path>/<model name>"
    engine = MLCEngine(model)
    for response in engine.chat.completions.create(
        messages=[{"role": "user", "content": "What is the meaning of life?"}],
        model=model,
        stream=True,
    ):
        for choice in response.choices:
            print(choice.delta.content, end="", flush=True)
    print("\n")

    engine.terminate()
    EOF

```

---
## Summary
- Build **MLC LLM** (CLI + Python) from the `mlc-llm` repo.
- Build **TVM Runtime only** from `mlc-ai/relax`.
- Install mlc_llm into your Python environment
- Install tvm into your Python environment.

This is the minimal working setup required for Orange Pi GPU‑accelerated MLC deployments.
