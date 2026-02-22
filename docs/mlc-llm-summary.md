

# Running MLC LLM on Mobile & Embedded Devices

Since 6/2025, I’ve been exploring how to run modern LLMs efficiently on mobile devices with limited Computation/Memory/power resources. The goal of the research is to learn how prompting can impact energy consumption. My approach centered around MLC LLM, a framework built on top of TVM that allows efficient inference on a wide range of devices

So far, I have run LLM models on below devices
- Google pixel 9
- Google pixel 8
- Orange PI 5 Pro

LLM models covered:
- XXXX
- XXXX
- XXX

I have gained consolidated understanding about the LLM model's performance, energy consumption with different prompts.

# Summary of learnings
## MLC LLM and pre-built package
pre-built package now is only avaialbe from https://mlc.ai/wheels/. You can only install pre-built package if there is a release matching your PC OS.
For example, if you are using Debian OS, you may not be able to have it installed.
If your PC is Windows, then you can easily install it, and start your Android App smoothly.
## Compile LLM models with mlc llm
For Android, you should pre-compile it, then run it.
For Linux OS to run Python API, you can do JIT compiling.
## Android Project upgrade for MLCChat
The example MLCChat is built on an older gradle version, it's required to upgrade it from xx to xx.

## Building TVM From Source for Android to fix an issue in TVM
I ran into this issue in Jan, 2026, when the TVM fix is not merged.

## Building TVM + MLC LLM From Source for Orange Pi 5 Pro
Feb, 2026, It's required to build these 2 becasue there are no pre-built package to use yet.
Something good to know which makes Orange PI 5 different:
- Ubuntu OS
- ARM
- OpenCL

## ✨ Project Setup
### Android MLCChat
- Windows with Android Studio
- Pre-built MLC LLM and TVM (Self-built Package if the pre-built version doesn't work)
- Run modern LLMs **fully offline** on Google Pixel 8 & 9

### Python API on Orange PI 5
- Build and optimize **MLC LLM + TVM** from source on Ubuntu running on Orange PI 5
- Run Modern LLMs **fully offline** on Orange PI 5

## 📦 Why Build From Source?

Pre-built MLC LLM packages are convenient, but:

| Platform | Pre-built MLC LLM availability | Issues |
|---------|-------------------|--------|
| **Windows** | ✔ Works well | No issues found |
| **Ubuntu ARM devices** | ❌ | Architecture mismatch, missing libs |
| **Debian** | ❌  | Not available  on 6/2025 |
| **Mac OS** | ❌ | Not available on 6/2025|

---

## 🔧 What I Built Myself
### Android MLCChat automation
It can run LLMs with 1000+ prompts with same environment temperature
### ✔ TVM (deep learning compiler)  
Fully custom build with:

- ARM64 target configuration  
- Vulkan options for devices that support it  
- Matching Python bindings  
- Optimized kernels for LLM workloads  

### ✔ MLC LLM (model compiler + runtime)  
Including:

- Python package  
- C++ runtime  
- Device‑specific build flags  
- Compiled model libraries (Llama, Mistral, etc.)

### ✔ Compiled LLM Models  
Optimized using MLC’s pipeline:

## 🟧 Running on Orange Pi 5 Pro

I’m currently focused on getting **MLC LLM running smoothly** on the **Orange Pi 5 Pro**, which features:

- Rockchip RK3588S  
- Strong ARM CPU performance  
- GPU & NPU (not fully supported by MLC yet)

Since **no pre-built MLC/TVM packages** exist for this board, I:

- Built TVM from source  
- Built MLC LLM from source  
- Compiled all required model libraries  
- Tested inference using Python + CLI runners  

Everything now works correctly on the Orange Pi 5 Pro.

---

## 📘 Key Learnings

### 1. Pre-built packages are limited  
Many mobile/embedded devices require custom builds.

### 2. TVM source builds are crucial  
TVM generates optimized kernels tailored to your hardware.

### 3. MLC LLM must match your TVM build  
Mismatched builds produce cryptic runtime failures.

### 4. ARM SBCs (like Orange Pi) need careful tuning  
Compiler flags, OPENCL drivers, and Python wheel compatibility all matter.

### 5. Quantization makes LLMs viable  
4-bit and 8-bit models run surprisingly well on RK3588S.

---

## 🚦 Status

- ✔ TVM compiled successfully on ARM64  
- ✔ MLC LLM runtime built for Linux ARM  
- ✔ Model libraries generated  
- ✔ Inference runs on Orange Pi 5 Pro  
- ⏳ Exploring GPU/NPU acceleration  
- ⏳ Packaging build scripts for public use  

---

## 🔮 Future Work

---

## 🤝 Contributing


---

## 📫 Contact

If you have questions or want to exchange ideas, feel free to reach out or open a discussion in the repo.

