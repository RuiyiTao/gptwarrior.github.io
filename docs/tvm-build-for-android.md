
# Fixing TVM LLVM Verification Error in MLC LLM (`dbg_declare must be a pointer or int`)

In Jan, 2026, I upgraded my Windows environment for Android MLCChat, which requries re-install MLC LLM and TVM package. But the environment is broken for a known issue without a fix being merged yet. Therefor, I need to compile TVM by myself with the fix suggested on GIT HUB posts.

TVM LLVM verification error encountered when running:

```bash
mlc_llm package
```

If you see errors like:

```
tvm.error.InternalError: LLVM module verification failed
location of #dbg_declare must be a pointer or int
#dbg_declare(float %v_input_red_temp.v0, ...)
```

The root cause is: **the TVM version inside your conda env is incompatible with your mlc-llm version**, and contains a known LLVM codegen bug.

---

# ✅ Root Cause Summary

- `mlc_llm package` **does NOT use the TVM inside your MLC repo**.
- It uses the TVM installed in your **conda environment** through the package:

```
mlc-ai-nightly-cpu
```

- This bundled TVM contains a bug fixed in TVM upstream: <https://github.com/apache/tvm/issues/18585>.
- MLC LLM requires a **forked TVM version** from the `relax` repo, NOT Apache TVM.

Therefore, the only working fix is to **build TVM from the correct repo + correct commit**, apply the LLVM patch, and replace the broken TVM in your conda env.

---

# ✅ Quick Fix (Most Developers Only Need This)
The fix is easy, but the challenge is to build TVM with right configurations which can work with the pre-built MLC LLM. It requires buld parameter configurations.
## 1. Clone the correct TVM fork
MLC LLM uses its own forked TVM (NOT Apache TVM):

```bash
git clone https://github.com/mlc-ai/relax
cd relax
git checkout e36cdd512ab1e5f4e5dbd87851a7709ad93fb3ec  // This is the commit which matches the pre-built MLC LLM package.
```

## 2. Apply the official fix to `codegen_llvm.cc`
Patch from TVM issue: <https://github.com/apache/tvm/issues/18585>

Apply option **#2** to:
```
relax/src/target/llvm/codegen_llvm.cc
```

## 3. Build TVM using CMake

```bash
mkdir build && cd build
cmake ..   -DCMAKE_BUILD_TYPE=RelWithDebInfo   -DUSE_LLVM="llvm-config --ignore-libllvm --link-static"   -DHIDE_PRIVATE_SYMBOLS=ON   -DUSE_VULKAN=ON   -DUSE_CUDA=OFF   -DUSE_ROCM=OFF   -DUSE_METAL=OFF   -DUSE_OPENCL=OFF
make -j
```

## 4. Install TVM into your conda environment

```bash
cd ../python
pip uninstall -y mlc_ai_nightly_cpu  # removes broken TVM
pip install .
```

Confirm installation:

```python
import tvm
print(tvm.__file__)
```

It should point to your **relax build**, not conda’s package directory.

## 5. Use a fresh clone of mlc-llm
Old copies may not match the new TVM.

```bash
git clone https://github.com/mlc-ai/mlc-llm
```

Then rerun:

```bash
python -m mlc_llm package
```

---

# ❗ Common Symptoms & What They Mean

| Symptom | Meaning | Fix |
|--------|---------|------|
| `dbg_declare must be a pointer or int` | LLVM codegen bug in TVM | Apply patch + rebuild TVM |
| `mlc_llm not found` | PATH alias missing | Use `python -m mlc_llm` |
| TVM path points to conda env | Wrong TVM selected | Reinstall TVM from relax repo |
| Apache TVM build fails in mlc-llm | Wrong repo | Use `mlc-ai/relax` instead |

---

# 💡 Why Apache TVM Will Never Work for MLC LLM

MLC LLM uses a **custom compiler stack** in the Relax repo.

Apache TVM ≠ MLC TVM.

If you build from the Apache repo:
- it will build successfully
- **but it will not load inside mlc_llm**
- mlc_llm package will error immediately

Only `mlc-ai/relax` contains:
- Relax dialect
- MLC-specific codegen
- Mobile-friendly ops
- Additional kernel lowering passes

---

# 🎯 Final Checklist

Before running `mlc_llm package`, verify:

### TVM is installed from *your local relax build*
```bash
python - << 'EOF'
import tvm
print(tvm.__file__)
EOF
```
Should **NOT** be:
```
.../site-packages/tvm/
```

Should be something like:
```
.../relax/python/tvm/
```

### mlc-llm is a *fresh clone*
Older copies reference old TVM APIs.

### Environment variable `TVM_SOURCE_DIR` is irrelevant
`mlc_llm` does **not** use it.

---

# ✔ Summary
To fix `dbg_declare must be a pointer or int` TVM errors:
1. Clone `mlc-ai/relax` at the commit used by mlc-llm.  
2. Apply the upstream LLVM fix.  
3. Build TVM from source with Vulkan enabled.  
4. Install it into your conda environment.  
5. Use a fresh mlc-llm clone and run `python -m mlc_llm package`.

This resolves the LLVM verification failure and ensures full compatibility with the MLC LLM toolchain.

---

If you want, I can also generate:
- version with diagrams
- version with troubleshooting table
- a “quick-fix only” cheat sheet
- a multi-device version (Android, Linux ARM, Windows)

Just let me know.
