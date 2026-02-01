# AI Post-Mortem Report

**Team:** Entangled  
**Project:** GPU-Accelerated Quantum-Enhanced Optimization for MTS

---

## 1. The Workflow

We employed a multi-agent AI workflow to accelerate development:

| Agent | Purpose | Platform |
|-------|---------|----------|
| **Cursor AI (Claude)** | Primary coding agent - QAOA implementation, debugging, benchmarking | Cursor IDE |
| **GitHub Copilot** | Code completion and suggestions | VS Code/Cursor |

### Workflow Structure

1. **Architecture Phase**: Used Cursor AI to help design the QAOA-MTS hybrid workflow based on PRD requirements
2. **Implementation Phase**: Iterative coding with Cursor AI for QAOA circuit, benchmark scripts, and notebook cells
3. **Debugging Phase**: Cursor AI identified and fixed CUDA-Q kernel compilation errors
4. **Analysis Phase**: Cursor AI helped interpret benchmark results and frame findings

---

## 2. Verification Strategy

### Unit Tests for AI-Generated Code

We implemented comprehensive unit tests in the notebook (`run_all_tests()`) to catch AI hallucinations:

```python
def run_all_tests():
    test_energy_matches_naive()      # Verifies LABS energy calculation
    test_energy_symmetries()         # E(s) == E(-s) and E(s) == E(s[::-1])
    test_delta_for_flip_is_exact()   # Incremental update correctness
    test_tabu_search_returns_consistent_energy()
    test_get_interactions_counts_and_bounds()
    test_bruteforce_optimum_small_N()
    print("All Phase 1 tests passed.")
```

### Specific Verification Steps

1. **Energy Function Verification**
   - Cross-checked AI-generated `labs_energy()` against naive brute-force implementation
   - Verified symmetry properties: `E(s) == E(-s)` and `E(s) == E(s[::-1])`

2. **QAOA Circuit Verification**
   - Tested circuit compilation with small N (N=4) before scaling
   - Verified sampled bitstrings have correct length and map to ±1 sequences

3. **Benchmark Verification**
   - Ran multiple seeds to ensure reproducibility
   - Cross-checked that random MTS achieves known optimal energies for small N

---

## 3. The "Vibe" Log

### Win: AI Saved Hours Debugging CUDA-Q Kernel Errors

**Problem:** The QAOA circuit failed to compile with cryptic CUDA-Q errors:
```
CompilerError: CUDA-Q does not allow assignment to variable t captured from parent scope.
(offending source -> i, t, k, kt = (quad[0], quad[1], quad[2], quad[3]))
```

**AI Solution:** Cursor AI identified that CUDA-Q kernels don't support Python tuple unpacking. It transformed:
```python
# BROKEN - tuple unpacking not supported
for quad in G4:
    i, t, k, kt = quad[0], quad[1], quad[2], quad[3]
```

To the working pattern used elsewhere in the notebook:
```python
# FIXED - indexed access pattern
for e in range(len(G4)):
    a = G4[e][0]
    b = G4[e][1]
    c = G4[e][2]
    d = G4[e][3]
```

**Time Saved:** ~2-3 hours of manual CUDA-Q documentation research.

---

### Learn: Providing Context Improved Results

**Initial Problem:** Early prompts like "fix the QAOA error" produced generic suggestions.

**Improved Strategy:** We provided:
1. Full error traceback
2. Reference to working code patterns in the same notebook
3. Explicit constraints (e.g., "CUDA-Q kernel restrictions")

**Example Effective Prompt:**
> "The QAOA circuit fails with 'variable captured from parent scope' error. Look at how `trotterized_circuit` in the same notebook handles G2 and G4 iteration - it works. Apply the same pattern."

This led to immediate, correct fixes.

---

### Fail: AI Hallucinated Function Parameters

**Problem:** AI generated benchmark code that called `mts()` with an `initial_pop` parameter:
```python
best_s, best_E, pop, energies = mts(N, pop_size=pop_size, 
                                     generations=generations, 
                                     initial_pop=initial_pop,  # WRONG!
                                     seed=seed)
```

**Reality:** The actual `mts()` function in the notebook doesn't have an `initial_pop` parameter.

**How We Caught It:** Runtime error when executing the benchmark cell.

**Fix:** AI corrected itself by using the existing `qaoa_quantum_enhanced_mts()` function instead.

**Lesson:** Always verify AI-generated function calls match actual function signatures.

---

### Context Dump: Key Prompts Used

**Prompt 1: Initial QAOA Implementation**
> "Read the PRD and look at the notebook. See if the QAOA part is correct? Or any improvements you suggest? We are in Phase 2. Tell me which testing should I be doing."

**Prompt 2: Benchmark Setup**
> "Help me with batch scripts using GPUs please. I have the py script but want to run it in Jupyter Lab."

**Prompt 3: Results Interpretation**
> "Read again. Updated the file. At the end, it performed worst! [shared benchmark results]"

**Prompt 4: Framing Negative Results**
> "Yes please [help write up this finding for AI_REPORT.md]"

---

## 4. Technical Findings Summary

### Benchmark Results

| N | QAOA Energy | Random Energy | QAOA Time | Random Time |
|---|-------------|---------------|-----------|-------------|
| 20 | 26.0 | 26.0 | 10.87s | 5.28s |
| 25 | 36.0 | 36.0 | 19.66s | 7.28s |

### Key Findings

1. **QAOA provides better initial populations**
   - QAOA seeds: E=50-96
   - Random seeds: typically E=150+ (estimated)

2. **MTS is robust enough to converge from random initialization**
   - For N ≤ 25, both methods achieve the same optimal energy
   - MTS doesn't require quantum seeding for these problem sizes

3. **QAOA parameter optimization overhead is significant**
   - Adds 5-12 seconds per run
   - Not justified unless MTS struggles to converge

4. **Negative result is still a valid finding**
   - Documents the regime where classical MTS is sufficient
   - Suggests QAOA advantage may appear at N ≥ 40 (per literature)

---

## 5. Lessons for Future AI-Assisted Development

1. **Provide domain-specific context** - AI performs better when given examples from the same codebase
2. **Verify function signatures** - AI may hallucinate parameters that don't exist
3. **Test incrementally** - Run small tests (N=4) before scaling up
4. **Frame negative results positively** - AI helped interpret findings constructively
5. **Use AI for debugging** - Particularly effective for framework-specific errors (CUDA-Q)
