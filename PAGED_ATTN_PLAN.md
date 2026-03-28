# Paged Attention Implementation Plan

## Status: IN PROGRESS (CPU fallback needs fix)

## Goal

Integrate vLLM-style paged attention into llama.cpp for the Aurora Fractal project.

**Critical Requirements:**
- CPU-first implementation (get CPU working first, then CUDA)
- BitNet is the primary test model (fast on CPU)
- Clean 2D tensor architecture - user explicitly confirmed: "Definitely 2D"
- Full vLLM-style paged attention with block tables, slot mapping
- Clean architecture for Delphi integration

## Scope Note

Working in `external/llama.orig/` (not `external/vllm/`). This is a fork of llama.cpp with Aurora modifications.

## Current State

### Working
- Model loads successfully with `--paged-attn on`
- KV tensors created correctly: `ne=[640,256,1]` (correct 2D format)
- KV cache copy operations work (cpy_k, cpy_v)
- Inference runs to completion

### Broken
- **CPU paged attention produces incorrect output**
- Test prompt: "Write a C++ hello world program:"
- Expected: C++ hello world code
- Actual: Talks about text adventure games (wrong attention output)

### Root Cause

The CPU fallback attention kernel in `llama-paged-attn.cpp` has correct tensor dimensions but the attention computation may have indexing issues.

## Files Modified

| File | Changes |
|------|---------|
| `src/llama-context.cpp:314-316` | Added missing parameter copies |
| `src/llama-context.cpp:3182-3183` | Fixed struct field names |
| `src/llama-kv-cache.cpp` | Changed to 2D tensor format for paged mode |
| `src/llama-paged-attn.cpp` | CPU fallback attention kernel |

## Test Command

```powershell
cd D:/zPython/Aurora_Fractal/external/llama.orig
.\build-cpu-test\bin\llama-cli.exe -m models\BitNet-b1.58-2B-4T\ggml-model-i2_s.gguf -c 256 -p "Write a C++ hello world program:" -n 64 --temp 0 --paged-attn on
```

**IMPORTANT**: Use cmd_runner for test execution to prevent stdout corruption:
```powershell
# From repo root with cmd_runner
uv run cmd_runner.py start --terminal conhost -- powershell -c "cd D:/zPython/Aurora_Fractal/external/llama.orig; .\build-cpu-test\bin\llama-cli.exe -m models\BitNet-b1.58-2B-4T\ggml-model-i2_s.gguf -c 256 -p 'Write a C++ hello world program:' -n 64 --temp 0 --paged-attn on"
```

## Known Issues & Fixes Applied

### Issue 1: Struct field name mismatch (FIXED)
- `sampler` → `samplers`, `n_sampler` → `n_samplers` in `llama-context.cpp`

### Issue 2: Missing parameter copies (FIXED)
- `paged_attn`, `paged_block_size`, `paged_partition_size` added to cparams

### Issue 3: Debug log flood (FIXED)
- Changed `AURORA_LOG_ERROR` to `AURORA_LOG_DEBUG` for tensor creation logs in `llama-kv-cache.cpp`

## Next Steps for GLM

1. **Analyze CPU attention kernel** (`src/llama-paged-attn.cpp:135-297`)
   - Verify tensor indexing matches 2D format `[n_embd_gqa, kv_size]`
   - KV cache: `k[pos * n_embd_gqa + kv_h * head_size + d]`
   - Query: `q[token * n_heads * head_size + h * head_size + d]`
   - Check GQA head mapping: `kv_h = h / heads_per_group`

2. **Test with cmd_runner**
   - Use cmd_runner for ALL model runs to prevent stdout corruption
   - Check output against expected "hello world" program

3. **Compare with standard attention**
   - Run same prompt without `--paged-attn` flag
   - Output should match when paged attention works correctly

4. **Debug approach**
   - Add temporary debug output to a log file (not stdout)
   - Check attention weights and values at key positions
   - Verify softmax produces reasonable distributions

## Architecture Notes

### Tensor Format (2D)
- K/V cache: `[n_embd_gqa, kv_size, n_stream]` where `n_embd_gqa = head_size * n_kv_heads`
- For BitNet: `[640, 256, 1]` = `[128*5, 256, 1]` (5 KV heads, head_size=128)

### GQA (Grouped Query Attention)
- BitNet has 5 query heads per KV head (n_heads=5, n_kv_heads=5)
- For models where n_heads > n_kv_heads: `heads_per_group = n_heads / n_kv_heads`
- Query head `h` maps to KV head `kv_h = h / heads_per_group`

## References

- `AGENTS.md` - Coding rules and guidelines
- `src/llama-graph.cpp:2710-2800` - `build_attn_paged()` function
- `src/models/llama.cpp:100-108` - Attention path selection