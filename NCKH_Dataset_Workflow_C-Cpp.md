# WORKFLOW: Xây dựng Dataset C/C++ cho NCKH — Dự đoán lỗi phần mềm bằng LLM

> **Label nhị phân**: `buggy` (1) = có lỗi · `fixed` (0) = không lỗi / đã sửa  
> **Ngôn ngữ**: C (ưu tiên), C++ (phụ)  
> **Công cụ chính**: Google Colab + Google Drive + GitHub + Google Sheet  
> **Thời gian**: 3 tuần · 2 giờ/người/ngày · 4 người

---

## 📌 TỔNG QUAN CHIẾN LƯỢC

| Quyết định | Lựa chọn | Lý do |
|---|---|---|
| Ngôn ngữ | C | Cú pháp đơn giản hơn C++, tài liệu CVE/patch nhiều, dễ extract code |
| Label | Binary 1/0 | Đúng yêu cầu đơn giản, giảm chi phí gán nhãn |
| Nguồn chính | Git history repos C/C++ | Miễn phí, commit message có chứa thông tin lỗi |
| Số repos mục tiêu | 3–5 repos C | Đủ đa dạng nhưng không quá nhiều để xử lý |
| Định dạng output | JSONL: `{code, label, project, commit}` | Tương thích với fine-tuning (LoRA/QLoRA) |
| Công cụ | Colab + Sheet + Drive | Không cần setup local, ai cũng dùng được |

---

## 🗓️ TIMELINE TỔNG QUAN

```
TUẦN 0 — Setup & RQ
    ▸ Checkpoint 0: RQ xác nhận + dataset mẫu OK
            ↓
TUẦN 1 — Thu thập code từ GitHub
    ▸ Checkpoint 1: ≥500 code pairs chưa gán nhãn
            ↓
TUẦN 2 — Gán nhãn + xử lý + cân bằng lớp
    ▸ Checkpoint 2: Dataset đã gán nhãn hoàn chỉnh
            ↓
TUẦN 3 — Kiểm tra chất lượng + bàn giao
    ▸ Checkpoint 3: Dataset cuối + đề cương hoàn thiện
```

---

## 🏗️ TUẦN 0 — SETUP & CHỌN FOCUS (2–3 ngày)

### Mục tiêu
Cả nhóm hiểu đề tài, chọn RQ, setup công cụ, chạy thử dữ liệu mẫu.

| Ngày | Vai trò | Nhiệm vụ | Công cụ | Cần code? |
|---|---|---|---|---|
| T0.1 | **Cả nhóm** | Đọc tài liệu giới thiệu: Bug prediction là gì, LLM fine-tuning là gì | Notion/PDF | ❌ |
| T0.2 | **Cả nhóm** | Họp quyết định: Focus C hay C++? (Đề xuất: **C**) | Google Meet | ❌ |
| T0.3 | **TV1 (Lead)** | Tạo GitHub repo nhóm + Colab notebook (dùng template) | GitHub, Colab | ⚠️ Mức 1 |
| T0.4 | **TV4 (Organizer)** | Tạo Google Sheet theo dõi tiến độ + dataset | Google Sheet | ❌ |
| T0.5 | **Tất cả** | Chạy thử 20 samples để cả nhóm hiểu dữ liệu | Colab | ❌ |

### Outputs Tuần 0
- ✅ RQ viết sơ bộ (vd: *"LLM có thể phân loại mã C có lỗi vs không lỗi từ commit-level code pairs không?"*)
- ✅ Repository nhóm đã tạo
- ✅ Google Sheet sẵn sàng
- ✅ Cả nhóm đã xem ≥1 ví dụ code `buggy` → `fixed`

---

## 🗄️ TUẦN 1 — THU THẬP CODE TỪ GITHUB (5–7 ngày)

### Bước 1.1: Chọn Repos Mục tiêu (1 buổi)

Đề xuất 5 repos C phù hợp:

| Repo | Stars | Lý do chọn |
|---|---|---|
| [curl](https://github.com/curl/curl) | 34k⭐ | 25+ năm history, nhiều CVE đã patch rõ ràng |
| [wget2](https://github.com/rockdaboot/wget2) | 2k⭐ | Network tool, commits message rõ ràng |
| [json-c](https://github.com/json-c/json-c) | 1.5k⭐ | Library nhỏ, dễ parse, ít phụ thuộc |
| [openssl](https://github.com/openssl/openssl) | 23k⭐ | Nhiều CVEs có patch commit rõ ràng |
| [nmap](https://github.com/nmap/nmap) | 9k⭐ | Security tool, bug fix commits dễ lọc |

### Bước 1.2: Trích xuất Code Pairs (3–4 ngày)

**Logic đơn giản:**
```
For mỗi repo:
  1. Clone về Google Drive
  2. Lọc commits có "fix" hoặc "bug" trong message
  3. Với mỗi commit:
     - Lấy file TRƯỚC khi fix (parent commit)
     - Lấy file SAU khi fix (current commit)
     → Đây là 1 cặp: BUGGY(1) → FIXED(0)
```

#### 📋 Colab Cell Templates (Copy-Paste)

**Cell 1: Clone repo** (chỉ đổi URL)

```python
!git clone --depth=200 https://github.com/curl/curl.git /content/curl
print("✅ Cloned curl")
```

**Cell 2: Lọc commits có "fix" hoặc "bug"**

```python
import subprocess, re

def get_fix_commits(repo_path):
    result = subprocess.run(
        ['git', '-C', repo_path, 'log', '--all',
         '--grep=fix', '--grep=bug', '--grep=repair',
         '--grep=patch', '--format=%H|%s|%ci', '--no-merges'],
        capture_output=True, text=True
    )
    commits = []
    for line in result.stdout.strip().split('\n'):
        if '|' in line:
            h, msg, date = line.split('|', 2)
            commits.append({'hash': h, 'msg': msg.lower(), 'date': date})
    return commits

fix_commits = get_fix_commits('/content/curl')
print(f"🔍 Found {len(fix_commits)} fix-related commits")
```

**Cell 3: Extract code pairs từ mỗi commit**

```python
import subprocess, json

def get_files_changed(repo_path, commit_hash):
    """Trả về danh sách file đã thay đổi trong commit"""
    result = subprocess.run(
        ['git', '-C', repo_path, 'diff-tree', '--no-commit-id', '-r',
         '--name-only', f'{commit_hash}^1', commit_hash],
        capture_output=True, text=True
    )
    return [f for f in result.stdout.split('\n') if f.strip()]

def get_file_content(repo_path, commit_hash, file_path, is_before=True):
    """Lấy nội dung file: before=True → parent, False → current"""
    ref = f'{commit_hash}^1' if is_before else commit_hash
    result = subprocess.run(
        ['git', '-C', repo_path, 'show', f'{ref}:{file_path}'],
        capture_output=True, text=True
    )
    return result.stdout if result.returncode == 0 else None

# Extract tất cả code pairs
samples = []
ext = {'.c', '.h'}  # file C — đổi sang '.cpp' nếu dùng C++

for c in fix_commits[:200]:  # giới hạn 200 commits mỗi repo
    files = get_files_changed('/content/curl', c['hash'])
    for f in files:
        if not any(f.endswith(e) for e in ext):
            continue
        buggy_code = get_file_content('/content/curl', c['hash'], f, is_before=True)
        fixed_code = get_file_content('/content/curl', c['hash'], f, is_before=False)
        if buggy_code and fixed_code and len(buggy_code) > 50:
            samples.append({
                'project': 'curl',
                'file': f,
                'commit': c['hash'],
                'buggy': buggy_code,
                'fixed': fixed_code,
                'label': 1  # 1 = có lỗi (code TRƯỚC khi fix)
            })

print(f"✅ Extracted {len(samples)} code pairs")
```

**Cell 4: Save to Google Drive**

```python
import json, os

os.makedirs('/content/drive/MyDrive/NCKH_dataset', exist_ok=True)
with open('/content/drive/MyDrive/NCKH_dataset/curl_raw.json', 'w') as f:
    json.dump(samples, f, ensure_ascii=False)
print(f"💾 Saved {len(samples)} samples to Drive")
```

**Cell 5: Tổng kết số lượng mẫu**

```python
import glob, json

all_files = glob.glob('/content/drive/MyDrive/NCKH_dataset/*_raw.json')
total = 0
for f in all_files:
    with open(f) as fh:
        data = json.load(fh)
        print(f"  {os.path.basename(f)}: {len(data)} samples")
        total += len(data)
print(f"\n📊 Total raw samples: {total}")
```

### Phân công Tuần 1

| Vai trò | TV | Nhiệm vụ | Cần code? |
|---|---|---|---|
| **Repo Runner A** | TV1 | Chạy Cell 1–4 cho repos 1–2 (curl, wget2) | ⚠️ Mức 1 |
| **Extraction Engineer** | TV2 | Chạy Cell 3–5, xử lý lỗi nhỏ (file rỗng, encoding lỗi) | ✅ Mức 2 |
| **Repo Runner B** | TV3 | Chạy cho repos 3–4 (json-c, openssl) | ⚠️ Mức 1 |
| **Metadata Recorder** | TV4 | Điền metadata vào Google Sheet sau mỗi batch 50 samples | ❌ |

### Checkpoint 1 (giữa tuần)

- ✅ Có **≥ 500 code pairs** đã extract
- ✅ Google Sheet có đầy đủ: `project`, `file_path`, `commit_hash`, `date`
- ✅ Cả nhóm đã xem 5 samples thử để hiểu format dữ liệu

---

## 🏷️ TUẦN 2 — GÁN NHÃN + XỬ LÝ + CÂN BẰNG (5–7 ngày)

### Bước 2.1: Gán nhãn Binary (2–3 ngày)

**Lưu ý quan trọng**: Vì dùng git history lọc "fix/bug" commits, code TRƯỚC khi fix đã là `buggy`(1), code SAU khi fix là `fixed`(0). **Bước này chủ yếu là VERIFY**, không phải gán từ đầu.

#### Quy trình Verify:

| Vai trò | TV | Công việc | Thời gian |
|---|---|---|---|
| **Labeler A** | TV2 | Kiểm 200 samples: nhìn code → tick ✅ (đúng) hoặc ❌ (sai) | 3–4 buổi |
| **Labeler B** | TV3 | Kiểm 200 samples khác | 3–4 buổi |
| **Quality Lead** | TV4 | Khi 2 người khác nhau → quyếtết định cuối cùng | Xen kẽ |
| **Recorder** | TV1 | Điền `verified` + `label_final` vào Sheet | 30p/ngày |

**Công cụ gán nhãn đơn giản (không cần code):**

Tạo 1 Google Form duy nhất:
```
Câu 1: "Đoạn code này có lỗi hay không?"
  ○ Có lỗi (BUGGY)
  ○ Không có lỗi (FIXED)
  ○ Không chắc → bỏ qua

Câu 2: Link đến code (paste từ Sheet)
Câu 3: Tên bạn
```

### Bước 2.2: Làm sạch & Cân bằng lớp (1 ngày)

| Bước | Hành động | Tool | Code? |
|---|---|---|---|
| Loại bỏ code trùng (hash trùng) | Chạy script dedup | Colab | ⚠️ Mức 1 |
| Xóa samples bị lỗi (file binary, encoding lỗi) | Filter `file.endswith('.c') or '.h'` | Colab | ⚠️ Mức 1 |
| Cân bằng 50/50 buggy:fixed | `resample()` từ sklearn | Colab | ⚠️ Mức 1 |
| Ghi nhãn cuối cùng | 1 = buggy, 0 = fixed trong Sheet | Sheet | ❌ |

**Template cân bằng lớp** (copy vào Colab):

```python
import pandas as pd
from sklearn.utils import resample

df = pd.read_json('/content/drive/MyDrive/NCKH_dataset/labeled.json')

print("📊 Trước khi cân bằng:")
print(df['label'].value_counts())

buggy = df[df['label'] == 1]
fixed = df[df['label'] == 0]
n = min(len(buggy), len(fixed))

buggy_bal = resample(buggy, n_samples=n, random_state=42)
fixed_bal = resample(fixed, n_samples=n, random_state=42)
df_final = pd.concat([buggy_bal, fixed_bal]).sample(frac=1, random_state=42)

print("\n📊 Sau khi cân bằng:")
print(df_final['label'].value_counts())

df_final.to_json('/content/drive/MyDrive/NCKH_dataset/dataset_balanced.json')
print("💾 Saved balanced dataset")
```

### Bước 2.3: Tăng cường dữ liệu — Augmentation (1–2 ngày)

⚠️ **QUY TẮC**: Nếu augmentation quá phức tạp → BỎ QUA. Chất lượng > Số lượng.

#### Phương án A (ưu tiên): Augmentation đơn giản bằng Colab

| Kỹ thuật | Mô tả | Độ khó |
|---|---|---|
| Rename variables | Đổi tên biến tạm (`count` → `num`, `tmp` → `tmp2`) | Mức 1 |
| Thay đổi whitespace | Thêm/bớt khoảng trắng, xuống dòng | Mức 1 |
| Swap init | `int x = 0` ↔ `int x` | Mức 2 |
| Insert unused var | Thêm `int _unused = 0;` vào đầu hàm | Mức 2 |

**Template augmentation đơn giản** (rename variables + whitespace):

```python
import re, random

def rename_vars(code):
    """Đổi tên biến local ngẫu nhiên (đơn giản, có thể sai một số chỗ)"""
    found = re.findall(r'\b([a-z_][a-z0-9_]*)\b', code)
    unique = list(set(found))[:10]
    mapping = {v: f'v{i}' for i, v in enumerate(unique, 1)}
    for old, new in mapping.items():
        code = re.sub(r'\b' + old + r'\b', new, code)
    return code

def noise_whitespace(code):
    """Thay đổi whitespace nhẹ"""
    lines = code.split('\n')
    out = []
    for l in lines:
        if random.random() > 0.6:
            l = '  ' + l  # thụt lề bừa
        out.append(l)
    return '\n'.join(out)

def augment(code, n=3):
    """Tạo n biến thể từ 1 đoạn code"""
    results = []
    for _ in range(n):
        aug = rename_vars(code)
        aug = noise_whitespace(aug)
        results.append(aug)
    return results
```

#### Phương án B (fallback — thủ công):
- Mỗi thành viên: mở code trong Colab viewer → đổi tay tên biến → copy vào Sheet
- Mục tiêu: 50–100 samples/người/ngày
- Label giữ nguyên (buggy/fixed)

### Checkpoint 2 (cuối tuần)

- ✅ `label_final` đã điền hết (không còn null/empty)
- ✅ Không còn code trùng lặp
- ✅ Dataset đã cân bằng (hoặc đã ghi rõ tỉ lệ hiện tại)

---

## 🧪 TUẾN 3 — KIỂM TRA CHẤT LƯỢNG + BÀN GIAO (3–4 ngày)

| Ngày | Vai trò | Nhiệm vụ | Công cụ | Code? |
|---|---|---|---|---|
| T3.1 | **Tất cả** | Mỗi người kiểm tra ngẫu nhiên 50 samples (25 buggy, 25 fixed) | Colab | ❌ |
| T3.2 | **TV4** | Tính thống kê cuối cùng: số mẫu mỗi lớp, số repos, tỉ lệ lỗi | Google Sheet | ❌ |
| T3.3 | **TV1** | Chạy dedup cuối + kiểm tra encoding | Colab cell | ⚠️ Mức 1 |
| T3.4 | **TV4** | Ghi `dataset_stats.json` (template bên dưới) | ❌ |
| T3.5 | **Cả nhóm** | Viết đề cương 3 phần: (1) RQ, (2) Dataset, (3) Pipeline | Notion/Word | ❌ |

**Template `dataset_stats.json`:**

```json
{
  "total_samples": 1200,
  "buggy": 600,
  "fixed": 600,
  "balance_ratio": "50/50",
  "projects": ["curl", "wget2", "json-c"],
  "avg_code_length_chars": 850,
  "min_code_length_chars": 50,
  "max_code_length_chars": 3200,
  "augmented_samples": 300,
  "label_agreement_rate": "90%",
  "generated_date": "2025-08-XX",
  "language": "C"
}
```

### Checkpoint 3 (cuối tuần)

- ✅ `dataset_final.jsonl` đã lưu trên Drive
- ✅ `dataset_stats.json` hoàn chỉnh
- ✅ Đề cương nghiên cứu 3 phần đã viết xong

---

## 📊 GOOGLE SHEET TEMPLATE

| id | project | file_path | commit_hash | label_draft | labeler_a | labeler_b | verified | label_final | bug_type | augmented | notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 001 | curl | src/tool_cb_dbg.c | a1b2c3d | 1 | ✅ | ✅ | ✅ | 1 | NULL deref | No | |
| 002 | curl | src/tool_cb_dbg.c | e4f5g6h | 0 | ✅ | ✅ | ✅ | 0 | — | No | |
| 003 | openssl | crypto/bn/bn_lcl.h | i9j0k1l | 1 | ✅ | ❌ | ✅ | 1 | Buffer overflow | Yes | TV1 quyết |

---

## 📚 NGUỒN DỮ LIỆU C/C++ PHÙ HỢP

| Dataset | Nguồn | Mô tả | Link |
|---|---|---|---|
| **Promis** | CTU Prague | 12 chương trình C, bugs được xác nhận có patch | [promis.online](https://www.promis.online/) |
| **CVEFixes** | NIST NVD | ~2,800 C/C++ CVEs có commit patch | [nvd.nist.gov](https://nvd.nist.gov/) |
| **SARD Juliet** | DARPA/NIST | Synthetic C/C++ bug cases, 110k+ | [samate.nist.gov](https://samate.nist.gov/SARD/) |
| **IntroClass** | IntroCompSciC | C programs dạy sinh viên, có bug | [GitHub](https://github.com/IntroCompSciC/) |

> 💡 **Khuyến nghị thứ tự sử dụng:**
> 1. **SARD Juliet** (có sẵn, cấu trúc rõ) — dùng cho Tuần 1 nếu git extraction gặp khó
> 2. **Git history từ Promis repos** — dùng khi đã quen workflow
> 3. **CVEFixes** — bổ sung nếu cần nhiều samples hơn

---

## 💻 MẶT ĐỘ CODE CẦN THIẾT

```
MỨC 1 — Copy-paste + chạy cell có sẵn (học 1 buổi là làm được)
  ├── Clone repos
  ├── Chạy deduplication script
  ├── Chạy stats script
  ├── Chạy balance script
  └── Template augmentation cơ bản

MỨC 2 — Sửa script nhỏ (đổi biến, thay filter, fix lỗi nhỏ) (cần 3–5 buổi học Python)
  ├── Điều chỉnh filter commit
  ├── Sửa lỗi encoding/file lỗi
  ├── Chạy auto-labeling pipeline
  └── Tùy chỉnh augmentation script

MỨC 3 — Viết script từ đầu (KHÔNG YÊU CẦU cho NCKH này)
  └── Chỉ dành cho thành viên đã biết Python trước
```

### Phân công code:

| Thành viên | Mức code đề xuất | Vai trò phù hợp |
|---|---|---|
| Biết chút Python trước | Mức 2 | Extraction Engineer, Augmentation |
| Chưa biết Python | Mức 1 | Repo Runner, Labeler, Recorder |
| Chưa biết code | Mức 1 + việc không cần code | Metadata, Labeler, Stats |

---

## ⚠️ PHƯƠNG ÁN DỰ PHÒNG (FALLBACK)

### Nếu extraction tự động thất bại:

| Vấn đề | Phương án dự phòng | Thời gian thêm |
|---|---|---|
| Repo quá lớn, script chạy lâu | Giảm phạm vi: chỉ lấy 3 repos nhỏ + `--depth=100` | +1 ngày |
| Commit message không rõ ràng | Dùng keyword "fix/bug/repair/patch" rộng hơn | — |
| File C bị parse lỗi | Lọc thêm: chỉ lấy file ≤ 500 dòng, có `.c` hoặc `.h` | — |
| Script extract bị lỗi | **Thủ công**: Xem git diff trên GitHub browser, copy code thủ công | +2–3 ngày, chỉ cần Sheet |
| Không đủ samples (≥500) | Dùng SARD Juliet dataset có sẵn để bổ sung | +0 ngày (chỉ cần download) |

### Nếu augmentation quá phức tạp:

```
→ BỎ QUA augmentation.
→ Dataset ít nhưng chất lượng cao > Dataset nhiều nhưng nhiễu.
→ Dành thời gian đó để verify labels kỹ hơn.
```

---

## ⚠️ RỦI RO THƯỜNG GẶP & CÁCH PHÒNG TRÁNH

| Rủi ro | Mức độ | Phòng tránh sớm |
|---|---|---|
| **Label sai nhiều** (>30%) | 🔴 Cao | Quy tắc 2/3 vote; kiểm tra ngẫu nhiên 20% sau mỗi buổi |
| **Mất cân bằng lớp** (90% fixed, 10% buggy) | 🔴 Cao | Cân bằng ngay sau khi gán nhãn xong (dùng template có sẵn) |
| **Code trùng lặp** (cùng file, commit gần nhau) | 🟡 TB | Chạy dedup script **mỗi buổi** thu thập |
| **Quá thời gian** (không kịp 500 samples) | 🔴 Cao | Focus 1 ngôn ngữ (C), 3 repos lớn — không lan man |
| **Mâu thuẫn nhóm** về label | 🟡 TB | TV1 Lead quyếtết định cuối cùng; lý lẽ dựa trên commit diff |
| **Thành viên nghỉ/ốm** | 🟡 TB | Mỗi người có backup; task công khai trên Sheet |
| **Colab bị ngắt / hết RAM** | 🟡 TB | Backup data mỗi buổi lên Drive; chia nhỏ batch xử lý |
| **Bug type quá đa dạng** → khó phân tích | 🟡 TB | Giới hạn 3–5 loại bug phổ biến (NPD, buffer overflow...) |
| **Snippet quá ngắn** (<20 dòng) | 🟡 TB | Quy tắc tối thiểu 50 ký tự — lọc ngay khi extract |
| **Emacs/Vim khác nhau** (git diff output khác nhau) | 🟡 TB | Dùng `git diff -U0` để lấy unified diff đồng nhất |

---

## 🗺️ WORKFLOW VISUAL SUMMARY

```
┌─────────────────────────────────────────────────────────────────┐
│  TUẦN 0: SETUP                                                 │
│  T0.1  Cả nhóm đọc tài liệu                                     │
│  T0.2  Họp: chọn C (hoặc C++), chọn 5 repos                    │
│  T0.3  TV1 tạo GitHub repo + Colab notebook                     │
│  T0.4  TV4 tạo Google Sheet + Drive folder                      │
│  T0.5  Cả nhóm chạy thử 20 samples                              │
└──────────────────────────┬──────────────────────────────────────┘
    ▸ CHECKPOINT 0: RQ + Công cụ đã sẵn sàng
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  TUẤN 1: THU THẬP                                              │
│  TV1+TV3: Clone repos → chạy extraction cells                   │
│  TV2: Xử lý lỗi, đảm bảo script chạy được                       │
│  TV4: Điền metadata vào Sheet sau mỗi batch 50                  │
└──────────────────────────┬──────────────────────────────────────┘
    ▸ CHECKPOINT 1: ≥500 code pairs
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  TUẤN 2: GÁN NHÃN + XỬ LÝ                                      │
│  Bước 2.1: Auto-label (git history) + Human verify (Form/Sheet) │
│  Bước 2.2: Dedup + làm sạch + cân bằng lớp (50/50)             │
│  Bước 2.3: Augmentation (rename vars + whitespace)              │
└──────────────────────────┬──────────────────────────────────────┘
    ▸ CHECKPOINT 2: Dataset đã gán nhãn + cân bằng
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  TUẤN 3: CHẤT LƯỢNG + BÀN GIAO                                 │
│  T3.1–3.2: Review 10% mẫu + tính stats                         │
│  T3.3–3.4: Dedup cuối + ghi dataset_stats.json                  │
│  T3.5: Viết đề cương 3 phần (RQ + Dataset + Pipeline)           │
└──────────────────────────┬──────────────────────────────────────┘
    ▸ CHECKPOINT 3: Bàn giao hoàn tất
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  📁 dataset_final.jsonl              ← Dataset cuối              │
│  📁 dataset_stats.json               ← Thống kê đầy đủ          │
│  📁 dataset_balanced.json            ← Đã cân bằng 50/50         │
│  📄 đề_cương_nc_kh.docx / .md        ← 3 phần: RQ+Dataset+Pipeline│
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 CHECKLIST BẮT ĐẦU NGAY (NGÀY ĐẦU TIÊN)

- [ ] **Cả nhóm**: Đọc Notion tab "Hiểu lĩnh vực" (bug prediction + LLM basics)
- [ ] **Cả nhóm**: Họp quyết định — chọn **C** làm ngôn ngữ chính
- [ ] **TV4**: Tạo Google Sheet (copy template ở phần trên)
- [ ] **TV4**: Tạo folder `NCKH_dataset` trên Drive
- [ ] **TV1**: Tạo GitHub repo nhóm (private hoặc public đều được)
- [ ] **TV1**: Mở Colab, tạo notebook mới, paste Cell 1–5 templates
- [ ] **Tất cả**: Chạy thử Cell 1–2 với 1 repo nhỏ (`json-c`) để thấy output

> 💡 **Mẹo cho ngày đầu**: Nếu chỉ làm được 4–5 việc trên là đủ — lúc này là lúc học tool, hiểu đề tài, không phải chạy tốc độ.

---

## 📝 LƯU Ý QUAN TRỌNG CHO NGƯỜI MỚI

1. **Không cần biết C để làm dataset** — chỉ cần đọc hiểu đủ để tick ✅/❌ khi verify
2. **Không cần compile/run code** — dataset là text thuần, không cần chạy chương trình
3. **Colab miễn phí đủ dùng** — không cần GPU cho bước thu thập + gán nhãn
4. **Code mẫu đã có sẵn** — nhiệm vụ chính là copy + chạy, không phải viết từ đầu
5. **Backup mỗi buổi** — save Drive mỗi ngày, không để mất dữ liệu
6. **Hỏi nhau** — nếu gặp lỗi script, hỏi trong nhóm trước khi mất 1 buổi debug

---

*Tài liệu này được tạo cho nhóm NCKH — đề tài "Ứng dụng LLM trong dự đoán lỗi phần mềm từ mã nguồn"*
