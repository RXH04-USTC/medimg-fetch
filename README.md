# medimg-fetch

An agent skill that enables AI assistants to **discover, evaluate, and download** open medical imaging datasets through natural language. Works with any agent framework that supports skill/tool loading — Cursor, OpenAI Codex, Claude, custom agents, etc.

Backed by a structured index of **1,100+ datasets** derived from [Project Imaging-X](https://github.com/uni-medical/Project-Imaging-X), the most comprehensive survey of open-access medical imaging datasets.

## What This Is

This is NOT a standalone application. It is an **agent skill** — a set of structured instructions + data that an AI agent reads to gain a new capability. When loaded, the agent can:

- Search 1,100+ medical imaging datasets by modality, anatomy, task, disease, or platform
- Classify access type (open / registration / application) and provide auth instructions
- Download open-access datasets on behalf of the user
- Run a supplementary web search to catch datasets not yet in the index

## Coverage

| Category | Count | Category | Count |
|----------|-------|----------|-------|
| 3D CT | 261 | Fundus Photography | 71 |
| 3D MR | 231 | X-ray | 56 |
| Histopathology | 114 | Endoscopy | 39 |
| Video | 75 | CT (2D) | 36 |
| 3D PET | 65 | Microscopy | 34 |
| 3D Ultrasound | 27 | 3D Other (DSA, OCT, CBCT) | 24 |
| Others | 23 | MRI (2D) | 22 |
| Ultrasound (2D) | 21 | OCT | 19 |
| Dermoscopy | 16 | PET (2D) | 11 |
| Infrared | 6 | **Total** | **1,151** |

Access breakdown: 🟢 Open 439 · 🟡 Registration 395 · 🔴 Application 7 · ⚪ Unknown 310

## Quick Start

### 1. Get the files

```bash
git clone https://github.com/<your-username>/medimg-fetch.git
```

### 2. Load into your agent

The entry point is `SKILL.md`. How you load it depends on your agent framework:

| Agent / Framework | How to load |
|---|---|
| **Cursor** | Symlink this folder into `~/.cursor/skills/` or `<project>/.cursor/skills/` |
| **OpenAI Codex** | Place this folder under `~/.codex/skills/` or reference `SKILL.md` in your agent config |
| **Claude (MCP / tool-use)** | Include `SKILL.md` as a system prompt appendix, or point a file-reading tool at it |
| **Custom agent** | Have your agent read `SKILL.md` at startup; it will know to load `datasets.json` and `access_rules.json` from the same directory |

### 3. Talk to it

```
帮我找所有公开的肾脏肿瘤影像数据集，不限模态
```
```
I need brain MRI segmentation datasets for glioma, preferably multi-center
```
```
找国内平台（天池、和鲸）上的胸部CT数据集
```
```
有没有消化道息肉的内镜视频数据集？
```

## How It Works

```
User query (natural language)
        │
        ▼
┌─────────────────────┐
│  1. Load Index      │  datasets.json — 1,100+ records, 14 fields each
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  2. Parse Query     │  Extract modality, anatomy, task, disease, etc.
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  3. Filter & Rank   │  Substring match on fields; rank by label, size, year
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  4. Present Results │  Markdown table with 🟢🟡🔴⚪ access tags
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  5. Web Search      │  One targeted search if index coverage < 5 (optional)
│     (gap-filling)   │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  6. Download Assist │  🟢 open → execute download (with user consent)
│                     │  🟡 registration → show auth steps
│                     │  🔴 application → show application process
└─────────────────────┘
```

## File Structure

```
medimg-fetch/
├── SKILL.md              # Core instructions for the agent
├── datasets.json         # Structured index (1,100+ datasets, 14 fields)
├── access_rules.json     # 25 platform rules: URL pattern → access type + auth guide
├── scripts/
│   └── update_index.py   # Re-generate index from latest Project Imaging-X
└── README.md             # You are here
```

### Key files explained

**`SKILL.md`** — The agent reads this to understand its task. Contains the 6-step retrieval workflow, matching heuristics, output format, download decision tree, and safety rules. Any agent that can read a markdown file can use this.

**`datasets.json`** — Each record looks like:
```json
{
  "section": "3D CT Datasets",
  "name": "TotalSegmentator",
  "url": "https://totalsegmentator.com/",
  "year": "2022",
  "dim": "3D",
  "modality": "CT",
  "structure": "Full Body",
  "samples": "1204",
  "label": true,
  "task": "Seg",
  "diseases": "Varied pathologies",
  "access": "open",
  "download_method": "Visit the URL to check download options",
  "auth_instructions": null
}
```

**`access_rules.json`** — Maps URL patterns to access types. Covers Kaggle, Grand-Challenge, TCIA, Tianchi, Heywhale, Synapse, PhysioNet, OpenNeuro, Zenodo, Figshare, Hugging Face, ADNI, and more. Each rule includes platform-specific download commands and step-by-step auth instructions.

## Updating the Index

```bash
# From GitHub (requires internet)
python scripts/update_index.py --github

# From a local clone of Project-Imaging-X
python scripts/update_index.py /path/to/Project-Imaging-X/README.md
```

This re-parses the source README and regenerates `datasets.json` with access classification.

## Acknowledgments

Dataset index derived from [Project Imaging-X](https://github.com/uni-medical/Project-Imaging-X). If you use this in your research, please cite:

```bibtex
@misc{projectimagingx2025,
  title={Project Imaging-X: A Survey of 1000+ Open-Access Medical Imaging Datasets
         for Foundation Model Development},
  author={Project Imaging-X Contributors},
  year={2025},
  url={https://github.com/uni-medical/Project-Imaging-X}
}
```

## License

MIT
