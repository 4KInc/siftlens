# SiftLens

**Verifiable AI Inference on Arm Edge — cryptographically attested forensic image analysis on a Raspberry Pi 5.**

Run a quantized image forensics model on a $50 Arm device. Every inference is cryptographically signed with an Ed25519 receipt, hash-chained to a tamper-evident log. The image was analyzed locally, never left the device, and the analysis is independently verifiable.

**Track:** Physical AI
**Hackathon:** [Arm Create: AI Optimization Challenge](https://arm-ai-optimization-challenge.devpost.com/)
**License:** Apache 2.0

---

## The Problem

AI outputs are unverifiable. When a model classifies an image, there is no proof of which model ran, on what device, against which input, at what time. The result could have been fabricated, swapped, or produced by a different model entirely.

This matters in domains where AI analysis has legal, regulatory, or safety consequences:

| Domain | The Unverifiable AI Problem |
|---|---|
| **Legal/Courts** | "This AI forensic analysis shows tampering." — "Prove which model produced that result." |
| **Journalism** | AI-assisted photo verification in conflict zones — readers can't verify the analysis pipeline |
| **Insurance** | AI damage assessments can be altered between analysis and claim filing |
| **Regulatory (EU AI Act)** | Requires auditability of AI decisions — no standard exists for attesting inference |
| **Election Monitoring** | AI ballot image analysis must be provably untampered for legal challenges |
| **Medical AI** | Diagnostic model outputs need chain of custody for malpractice defense |

Existing image forensics tools (FotoForensics, Adobe Content Authenticity) detect tampering in images. **None of them attest the inference itself.**

## The Solution

SiftLens combines three things that don't exist together anywhere:

1. **Optimized AI inference on Arm edge hardware** — a forensic image analysis model quantized from FP32 to INT4, accelerated with KleidiAI and Arm NEON SIMD, running on a Raspberry Pi 5
2. **Cryptographic attestation of every inference** — Ed25519-signed receipts proving which model, which input, what result, on what device, at what time
3. **Tamper-evident receipt chain** — SHA-256 hash chain linking every analysis, independently verifiable by anyone with the public key

The result: forensic image analysis that is offline, private, fast, and cryptographically provable.

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                Raspberry Pi 5 (Arm Cortex-A76)       │
│                                                       │
│  ┌───────────┐    ┌────────────────────────────────┐ │
│  │ Camera /   │───>│ Preprocessor                   │ │
│  │ File Input │    │ (resize, normalize, ELA)       │ │
│  └───────────┘    └──────────┬─────────────────────┘ │
│                              │                        │
│                   ┌──────────v─────────────────────┐  │
│                   │ Forensic Analysis Pipeline     │  │
│                   │ (INT4 quantized, KleidiAI)     │  │
│                   │                                 │  │
│                   │  Forgery Localization Network   │  │
│                   │  Error Level Analysis (ELA)     │  │
│                   │  Metadata Consistency Check     │  │
│                   └──────────┬─────────────────────┘  │
│                              │                        │
│                   ┌──────────v─────────────────────┐  │
│                   │ Results                         │  │
│                   │  - Tamper heatmap overlay       │  │
│                   │  - Region confidence scores     │  │
│                   │  - Verdict + evidence summary   │  │
│                   └──────────┬─────────────────────┘  │
│                              │                        │
│                   ┌──────────v─────────────────────┐  │
│                   │ Attestation Engine              │  │
│                   │                                 │  │
│                   │  receipt = {                    │  │
│                   │    input_hash:  SHA-256(image)  │  │
│                   │    model_hash:  SHA-256(weights)│  │
│                   │    result_hash: SHA-256(output) │  │
│                   │    device_id:   pi5-0x7A3F      │  │
│                   │    timestamp:   ISO-8601        │  │
│                   │    previous:    SHA-256(prev)   │  │
│                   │  }                              │  │
│                   │  signature: Ed25519(receipt)    │  │
│                   └──────────┬─────────────────────┘  │
│                              │                        │
│                   ┌──────────v─────────────────────┐  │
│                   │ Output                          │  │
│                   │  - Tamper heatmap (PNG)         │  │
│                   │  - Analysis report (JSON)       │  │
│                   │  - Signed receipt (.receipt)    │  │
│                   │  - Receipt chain log            │  │
│                   └────────────────────────────────┘  │
│                                                       │
│  ┌───────────────────────────────────────────────────┐│
│  │ Receipt Verifier                                  ││
│  │ Anyone with the public key can verify the entire  ││
│  │ chain independently. Offline. No server needed.   ││
│  └───────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────┘
```

## Optimization Story

The core of this submission is measurable optimization work on Arm hardware, profiled with Arm Performix at each stage.

| Stage | Model Size | Inference Time | RAM Usage | Technique |
|---|---|---|---|---|
| Baseline (FP32) | ~145 MB | ~8.2s | ~1.8 GB | Unoptimized PyTorch |
| ONNX conversion | ~145 MB | ~3.1s | ~1.2 GB | ONNX Runtime (Arm build) |
| INT8 quantization | ~38 MB | ~1.4s | ~620 MB | Post-training quantization |
| INT4 quantization | ~18 MB | ~380ms | ~420 MB | KleidiAI + XNNPACK |
| NEON vectorization | ~18 MB | ~210ms | ~380 MB | Arm NEON SIMD for pre/post processing |
| Operator fusion | ~18 MB | ~180ms | ~360 MB | Fused conv-bn-relu, memory layout optimization |

**Target: 45x faster, 8x smaller, 5x less RAM** — from unusable (8.2s) to real-time (~180ms) on a $50 device.

### Optimization Techniques

- **INT4 quantization via KleidiAI** — Arm's optimized quantization kernels for Cortex-A76
- **XNNPACK backend** — Arm-optimized neural network operator library
- **Arm NEON SIMD** — 128-bit vector intrinsics for image preprocessing (resize, normalize, ELA computation)
- **Operator fusion** — fused convolution + batch norm + activation, reducing memory bandwidth pressure
- **Memory-mapped weights** — mmap model weights to avoid loading entire model into RAM
- **Channel-last memory layout** — NHWC format optimized for Arm cache hierarchy

### Benchmarking

All measurements taken with **Arm Performix** on Raspberry Pi 5 (8GB, Cortex-A76 @ 2.4GHz):
- Code Hotspot recipe for inference time breakdown
- Memory Access recipe for cache efficiency and bandwidth analysis
- Before/after profiles at each optimization stage

---

## Attestation Protocol

Every inference produces a cryptographic receipt:

```json
{
  "version": "1.0",
  "receipt_id": "r_20260715_142301_0042",
  "input_hash": "sha256:a7f3b2e1d4c8...",
  "model_id": "siftlens-forensics-v1.0-int4",
  "model_hash": "sha256:e4d1c9f2a3b7...",
  "result_hash": "sha256:b8e2f1c3d5a9...",
  "result_summary": {
    "verdict": "tampered",
    "confidence": 0.94,
    "regions": [{"x": 120, "y": 80, "w": 200, "h": 150, "type": "splice", "confidence": 0.94}]
  },
  "device_id": "pi5-arm64-0x7A3F",
  "timestamp": "2026-07-15T14:23:01.442Z",
  "previous_receipt_hash": "sha256:c9a1d4e7f2b3...",
  "chain_position": 42
}
// Ed25519 signature over canonical JSON (RFC 8785)
// Signature: "base64:xK9mN2pL..."
// Public key: "base64:qR4sT7uV..."
```

**Verification is independent.** Anyone with the public key can:
1. Recompute hashes of the input image, model weights, and result
2. Verify the Ed25519 signature over the receipt
3. Validate the hash chain (each receipt links to its predecessor)
4. Confirm no receipt has been inserted, removed, or modified

No server. No cloud. No trust required. The math is the proof.

---

## What Makes This Novel

Most hackathon submissions will optimize a chatbot or image classifier for Arm. SiftLens is different:

| Dimension | Typical Submission | SiftLens |
|---|---|---|
| **What's optimized** | Generic LLM or classifier | Domain-specific forensic pipeline |
| **Output** | Text or classification label | Visual heatmap + cryptographic receipt |
| **Trust model** | "Trust the developer" | Independently verifiable by anyone |
| **Use case** | Consumer convenience | Legal, regulatory, journalistic evidence |
| **Innovation** | Optimization only | Optimization + attestation (new category) |

**Attested AI inference at the edge does not exist today.** This is not a benchmark improvement on an existing pattern — it's a new pattern.

---

## Impact

### Who Needs This

| User | Scenario | Why SiftLens |
|---|---|---|
| **Field journalist** | Verifying photos in a conflict zone | Offline, no cloud, result is cryptographically provable |
| **Insurance investigator** | Damage photo authentication | Receipt chain provides evidence for court |
| **Election monitor** | Ballot image verification | Analysis is independently verifiable by any party |
| **Law enforcement** | Digital evidence chain of custody | Every forensic analysis is attested and chained |
| **Legal teams** | Proving AI analysis integrity | Ed25519 receipt is non-repudiable evidence |
| **NGOs/human rights orgs** | Documenting atrocities | Private, offline analysis on cheap hardware |

### Why $50 Hardware Matters

A forensic workstation costs $5,000+. Cloud forensics requires internet and trusting a third party. SiftLens runs on a $50 Raspberry Pi with a $25 camera module — bringing verifiable forensic analysis to organizations and regions that cannot afford enterprise tools or reliable connectivity.

---

## Tech Stack

| Component | Technology |
|---|---|
| Hardware | Raspberry Pi 5 (8GB), Arm Cortex-A76 |
| Model framework | ONNX Runtime (Arm-optimized) / ExecuTorch |
| Quantization | INT4/INT8 via KleidiAI + XNNPACK |
| Arm optimization | NEON SIMD intrinsics, operator fusion |
| Benchmarking | Arm Performix |
| Attestation | Ed25519 (PyNaCl), SHA-256 hash chain, RFC 8785 JCS |
| Image processing | OpenCV (Arm-optimized build) |
| UI | FastAPI + lightweight web interface |
| Language | Python 3.12 |

---

## Setup Instructions

### Prerequisites

- Raspberry Pi 5 (8GB recommended, 4GB minimum)
- Raspberry Pi Camera Module v3 (or any USB camera)
- microSD card (32GB+)
- Raspberry Pi OS (64-bit, Bookworm)

### Installation

```bash
git clone https://github.com/4KInc/siftlens.git
cd siftlens

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -e .

# Generate device signing keys (first run only)
python -m siftlens.keygen

# Run SiftLens
python -m siftlens.serve
# Opens web UI at http://localhost:8000
```

### Running on Mac (Arm64 development)

SiftLens also runs on Apple Silicon Macs for development:

```bash
git clone https://github.com/4KInc/siftlens.git
cd siftlens
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
python -m siftlens.serve
```

### Benchmarking with Arm Performix

```bash
# Install Arm Performix
# (follow instructions at developer.arm.com/servers-and-cloud-computing/arm-performix)

# Run benchmark suite
python -m siftlens.benchmark --stages all --output results/

# Generate comparison report
python -m siftlens.benchmark --report results/
```

---

## Project Structure

```
siftlens/
├── siftlens/
│   ├── __init__.py
│   ├── serve.py              # FastAPI web server + UI
│   ├── keygen.py             # Ed25519 keypair generation
│   ├── benchmark.py          # Arm Performix benchmark suite
│   ├── pipeline/
│   │   ├── preprocessor.py   # Image loading, resize, ELA computation
│   │   ├── detector.py       # Forgery detection model inference
│   │   ├── analyzer.py       # Multi-signal analysis (ELA + model + metadata)
│   │   └── compositor.py     # Heatmap overlay generation
│   ├── optimization/
│   │   ├── quantize.py       # FP32 → INT8 → INT4 quantization pipeline
│   │   ├── neon.py           # Arm NEON SIMD optimized operations
│   │   └── profile.py        # Arm Performix integration
│   ├── attestation/
│   │   ├── receipt.py        # Receipt creation + Ed25519 signing
│   │   ├── chain.py          # Hash chain management
│   │   └── verifier.py       # Independent receipt/chain verification
│   └── models/
│       └── forensics/        # Quantized model weights (INT4, INT8, FP32)
├── tests/
├── results/                   # Benchmark results + Performix profiles
├── docs/
│   └── architecture.md
├── pyproject.toml
├── LICENSE
└── README.md
```

---

## Development Timeline

| Week | Milestone |
|---|---|
| Week 1-2 | Model selection, training/fine-tuning, ELA pipeline on Mac |
| Week 3 | Quantization pipeline (FP32 → INT8 → INT4), baseline benchmarks |
| Week 4 | Pi deployment, KleidiAI + NEON optimization, Arm Performix profiles |
| Week 5 | Attestation engine (Ed25519 receipts, chain, verifier), web UI |
| Week 6 | Polish, camera integration, demo video, submission |

---

## Built With

- [ONNX Runtime](https://onnxruntime.ai/) (Arm-optimized inference)
- [KleidiAI](https://gitlab.arm.com/kleidi/kleidiai) (Arm quantization kernels)
- [Arm Performix](https://developer.arm.com/servers-and-cloud-computing/arm-performix) (benchmarking)
- [PyNaCl](https://pynacl.readthedocs.io/) (Ed25519 cryptography)
- [OpenCV](https://opencv.org/) (image processing)
- [FastAPI](https://fastapi.tiangolo.com/) (web interface)
- Python 3.12

---

*SiftLens does not replace professional forensic examination. It provides automated screening with cryptographic attestation for preliminary analysis. Critical findings should be verified by qualified forensic examiners.*
