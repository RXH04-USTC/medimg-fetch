---
name: medimg-fetch
description: >-
  Search, recommend, and download open medical imaging datasets.
  Backed by a structured index of 1100+ datasets from Project-Imaging-X
  (the most comprehensive open medical imaging dataset survey).
  Use when the user asks for medical imaging datasets, download links,
  or dataset recommendations for specific modalities, anatomies, tasks,
  or diseases. Also use when the user mentions terms like CT, MRI, X-ray,
  pathology, endoscopy, ultrasound, fundus, PET, microscopy, or any
  medical imaging related dataset query.
---

# medimg-fetch — Medical Image Dataset Discovery & Download

## Data Source

This skill is backed by a structured JSON index (`datasets.json`) derived from
[Project-Imaging-X](https://github.com/uni-medical/Project-Imaging-X), a survey
of 1000+ open-access medical imaging datasets.

Each record has these fields:

| Field | Description | Examples |
|-------|-------------|----------|
| `section` | Category in the survey | 3D CT Datasets, Endoscopy, Video Datasets |
| `name` | Dataset name | TotalSegmentator, BraTS 2023, Kvasir-SEG |
| `url` | Official download / landing page | https://totalsegmentator.com/ |
| `year` | Publication year | 2022 |
| `dim` | Dimensionality | 2D, 3D, Video, 2D+Video |
| `modality` | Imaging modality | CT, MR, X-Ray, Endoscopy, Histopathology, US |
| `structure` | Anatomical region | Lung, Brain, Kidney, Full Body, Colon |
| `samples` | Number of images/volumes | 1204, 5.9k, 100k |
| `label` | Has annotations | true / false |
| `task` | AI task type | Seg, Cls, Det, Reg, Rec, VQA |
| `diseases` | Target diseases | Lung cancer, Glioma, Polyp, NA |
| `access` | Access type | open, registration, application, unknown |
| `download_method` | How to download | wget/curl, Kaggle API, NBIA Data Retriever |
| `auth_instructions` | Auth steps (null if open) | Step-by-step registration/application guide |

## Retrieval Workflow

When a user asks for medical imaging datasets, follow these steps:

### Step 1: Load the Index

Read the file `datasets.json` located in the same directory as this SKILL.md.

### Step 2: Understand the Query

Extract constraints from the user's natural language request:

- **Modality**: CT, MRI/MR, X-ray, ultrasound (US), endoscopy, pathology/histopathology/WSI, fundus, OCT, PET, microscopy, dermoscopy, infrared, mammography, DSA, CBCT
- **Anatomy/Structure**: brain, lung, liver, kidney, heart, breast, prostate, colon, retina, spine, etc.
- **Task**: segmentation (Seg), classification (Cls), detection (Det), registration (Reg), reconstruction (Rec), prediction (Pred), VQA, tracking
- **Dimension**: 2D, 3D, video
- **Disease**: specific disease names (COVID-19, glioma, Alzheimer's, polyp, etc.)
- **Other**: label availability, sample size, platform preference (Kaggle, TCIA, Tianchi, etc.)

### Step 3: Search and Filter

Filter the JSON records by matching user constraints against the corresponding fields. Use case-insensitive substring matching. A record matches if it satisfies ALL stated constraints.

**Matching tips:**
- For modality: `MR` matches both "MR" and "MRI"; `pathology` matches "Histopathology (Patch)" and "Histopathology (WSI)"
- For anatomy: `kidney` matches "Kidney", "Kidney, Lung" (multi-organ); `brain` matches "Brain"
- For task: `Seg` matches "Seg", "Seg, Cls", "Seg/Cls"
- For disease: use substring match — `glioma` matches "Glioma", "Low Grade Glioma", "Diffuse Gliomas"
- Some datasets appear in multiple sections (cross-listed); deduplicate by `name + url`

### Step 4: Rank Results

Prefer datasets that:
1. Have `label: true` (annotated)
2. Have more samples
3. Are more recent (higher year)
4. Have a working URL (non-empty `url`)

### Step 5: Present Results

Return results as a **markdown table** with access info:

```
| Dataset | Year | Modality | Anatomy | Samples | Task | Access | Link |
|---------|------|----------|---------|---------|------|--------|------|
| ... | ... | ... | ... | ... | ... | 🟢 open | [Link](url) |
| ... | ... | ... | ... | ... | ... | 🟡 registration | [Link](url) |
| ... | ... | ... | ... | ... | ... | 🔴 application | [Link](url) |
| ... | ... | ... | ... | ... | ... | ⚪ unknown | [Link](url) |
```

Access type icons:
- 🟢 **open**: direct download, no account needed
- 🟡 **registration**: free account required (Kaggle, Grand-Challenge, Tianchi, etc.)
- 🔴 **application**: formal application with approval process (ADNI, PPMI, etc.)
- ⚪ **unknown**: check the landing page manually

After the table, add:
- A brief summary of how many datasets matched and the main categories
- If the user asked about licensing/commercial use — state honestly that the index does not include license info and they should check each dataset's landing page
- If very few results match, suggest relaxing constraints and show near-matches

### Step 6: Supplementary Web Search (gap-filling)

The local index covers 1100+ datasets but may miss very recent or niche ones.
After presenting results from Step 5, do ONE targeted web search to fill gaps.

**Rules — keep it surgical, not broad:**
1. Do NOT search if the local index already returned ≥ 5 good matches.
2. Construct a narrow query that targets what the index MIGHT have missed.
   Use this template:
   ```
   "<specific_disease_or_anatomy>" "<modality>" dataset download site:kaggle.com OR site:zenodo.org OR site:huggingface.co OR site:grand-challenge.org <current_year>
   ```
   For example, if the user asked for "thyroid ultrasound segmentation" and
   the index returned only 2 results, search:
   `"thyroid" "ultrasound" segmentation dataset 2025 2026`
3. Limit to **1 web search call** (not multiple).
4. From the search results, only add datasets that are NOT already in
   your table (compare by name or URL).
5. Mark web-sourced additions clearly in the output:
   ```
   | Dataset | Year | ... | Access | Link | Source |
   | ...     | ...  | ... | ...    | ...  | 📚 索引 |
   | ...     | ...  | ... | ...    | ...  | 🔍 网络补充 |
   ```
6. For web-sourced datasets, the access type is unknown unless you can
   infer it from the URL pattern (apply the same `access_rules.json` logic).
7. If the web search finds nothing new, say "索引结果已较全面，网络补充搜索未发现额外数据集。"

**When to definitely search:**
- The index returned 0 results
- The query is for a very specific / rare disease or a very new modality (e.g., photoacoustic imaging)
- The user explicitly asks "有没有更新的" or "还有别的吗"

**When to skip:**
- The index returned ≥ 5 matches covering the query well
- The query is for a broad, well-covered area (e.g., "chest X-ray classification")

### Step 7: Download Assistance

After presenting results, offer to help download. Follow this decision tree:

**For `access: "open"` datasets:**
1. Tell the user: "以下数据集可直接下载，是否需要我帮你下载？" and list the open ones.
2. The user must explicitly confirm before you execute any download command.
   (Cursor's tool-use approval dialog serves as the permission gate — every Shell
   command the agent proposes is shown to the user who must click "Allow".)
3. Once confirmed, execute the appropriate download command based on `download_method`:
   - **GitHub**: `git clone <url>` or `wget <release_url>`
   - **Zenodo/Figshare/Mendeley**: `wget <direct_file_url>` (find the file link on the page first)
   - **Google Drive**: `pip install gdown && gdown <file_id>`
   - **Hugging Face**: `pip install huggingface_hub && huggingface-cli download <repo_id>`
   - **TCIA (cancerimagingarchive.net)**: explain NBIA Data Retriever or use the TCIA REST API
   - **OpenNeuro**: `aws s3 sync --no-sign-request s3://openneuro.org/<dsID> ./<dsID>/`
   - **Medical Decathlon**: `wget <google_drive_link>` via gdown
4. Download to the user's current working directory or a path they specify.
5. Show progress: for large files, use `wget --progress=bar` or equivalent.

**For `access: "registration"` datasets:**
1. Do NOT attempt to download. Instead, present the `auth_instructions` field clearly:
   ```
   ⚠️ 此数据集需要注册才能下载。步骤如下：
   <auth_instructions content>
   ```
2. If the user says they already have credentials (e.g., Kaggle API key configured),
   then proceed with the platform-specific download command:
   - **Kaggle**: `kaggle datasets download -d <id>` or `kaggle competitions download -c <name>`
   - **Synapse**: `synapse get <synID> -r`
   - **Grand-Challenge**: guide the user to the download page after login
   - **Tianchi/Heywhale/Baidu**: these usually require browser-based download; explain this honestly

**For `access: "application"` datasets:**
1. Do NOT attempt to download. Present the `auth_instructions` and emphasize:
   ```
   🔒 此数据集需要正式申请并经审批后才能获取。
   <auth_instructions content>
   预计审批时间：通常 1-4 周。
   ```

**For `access: "unknown"` datasets:**
1. Say: "该数据集的访问方式未能自动识别，请访问数据集主页确认下载方式。"
2. Offer to open/fetch the URL to check the page and determine access type at runtime.

**Download safety rules:**
- NEVER download to system directories. Default to CWD or user-specified path.
- NEVER store or transmit credentials. If Kaggle/Synapse credentials are needed, rely on the user's pre-configured CLI auth (~/.kaggle/kaggle.json, ~/.synapseConfig, etc.).
- For downloads > 10 GB, warn the user about disk space before proceeding.
- If a download fails, report the error clearly; do NOT retry silently.

## Updating the Index

If `datasets.json` is missing or the user wants the latest data, run:

```bash
python scripts/update_index.py --github
```

This fetches the latest README from GitHub and regenerates the JSON.

To update from a local clone:

```bash
python scripts/update_index.py /path/to/Project-Imaging-X/README.md
```

## Edge Cases

- **User asks for a dataset that doesn't exist in the index**: say so honestly; suggest the closest alternatives. Do NOT invent dataset names or URLs.
- **User asks about license/commercial use**: the index does not track this; advise checking the dataset's official page.
- **User asks for datasets on specific platforms** (e.g., Tianchi, Kaggle): filter by URL substring (e.g., `tianchi.aliyun.com`, `kaggle.com`, `grand-challenge.org`, `cancerimagingarchive.net`).
- **User wants the biggest datasets**: sort by `samples` (note: values like "100k", "1.4M" need parsing).
- **User wants only directly downloadable datasets**: filter by `access: "open"`.
- **User asks "help me download all of them"**: refuse bulk download without per-dataset confirmation. Process one at a time, confirming each.
- **Download target is a landing page, not a file**: some URLs point to project homepages. In that case, fetch the page to locate the actual download link before attempting wget/curl.
