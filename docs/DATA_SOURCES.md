# Data Sources - Nguồn Dữ Liệu Gốc

## 📚 AdaParse Dataset

Dữ liệu gốc được sử dụng để sinh benchmark khử trùng lặp.

### Tham khảo

- **Bài báo gốc**: Siebenschuh et al., 2025  
  - Tên: "AdaParse: An Adaptive Parallel PDF Parsing and Resource Scaling Engine"  
  - Conference: ML Systems (MLSys)  

- **Repository**: https://github.com/allenai/AdaParse

### Cấu trúc dữ liệu

```
AdaParse/
├── joint_to_html/parsed_pdfs/
│   ├── part-00000.jsonl
│   ├── part-00001.jsonl
│   └── ...
│
├── joint_to_pymupdf/parsed_pdfs/
│   ├── part-00000.jsonl
│   ├── part-00001.jsonl
│   └── ...
│
├── joint_to_nougat/parsed_pdfs/
│   ├── part-00000.jsonl
│   ├── part-00001.jsonl
│   └── ...
│
└── joint_to_tesseract/parsed_pdfs/
    ├── part-00000.jsonl
    ├── part-00001.jsonl
    └── ...
```

### Format dữ liệu (mỗi dòng JSONL)

```json
{
  "path": "arxiv/pdf/1352.4534v1.pdf",
  "text": "The quick brown fox jumps over the lazy dog...",
  "id": "1352.4534v1",
  "source": "arxiv"
}
```

### 4 Parser được sử dụng

| Parser | Công cụ | Mục đích |
|---|---|---|
| **html** | Web scraping | Trích xuất từ trang HTML gốc |
| **pymupdf** | PyMuPDF 1.23.0 | PDF parsing library Python |
| **nougat** | Nougat (Meta) | Neural OCR cho academic docs |
| **tesseract** | Tesseract 5.x | Traditional OCR engine |

---

## 🔄 Luồng xử lý Benchmark

### Bước 1: Sample từ mỗi parser

```python
# synthetic_benchmark/gen_dedup_benchmark.py
parser_sources = ['html', 'nougat', 'pymupdf', 'tesseract']
N_per_source = 4000  # mặc định

# Kết quả: ~4000 bài báo duy nhất từ mỗi parser
# (Có overlap do cùng PDF gốc)
```

**Output**: `data_list` (tài liệu gốc, `modification=0`)

### Bước 2: Collect cross-parser duplicates

```python
# synthetic_benchmark/dedup_benchmark_utils.py::collect_duplicates()

# Logic: Cùng PDF nhưng parse bằng công cụ khác
# Ví dụ:
#   PDF: arxiv/pdf/1352.4534v1.pdf
#   - Parser 1 (HTML): text_html
#   - Parser 2 (PyMuPDF): text_pymupdf → gần-trùng do parsing difference
```

**Output**: `data_dupl_list` (parser duplicates, `modification=1`)  
**Similarity**: Jaccard ~0.8-0.95 (gần-trùng, dễ detect)

### Bước 3: Create truncation duplicates

```python
# synthetic_benchmark/dedup_benchmark_utils.py::truncate()

# Cắt xén 1-20% nội dung từ các vị trí khác nhau
# Ví dụ:
#   Original: "The quick brown fox jumps over the lazy dog"
#   Truncate 20% từ giữa: "The quick brown fox jumps lazy dog"
```

**Output**: `data_trunc_list` (truncation duplicates, `modification=2`)  
**Similarity**: Jaccard ~0.7-0.9 (khó hơn để detect)

### Bước 4: Merge + Label

```python
# synthetic_benchmark/dedup_benchmark_utils.py::make_benchmark_dataframe()

# Merge 3 danh sách và tạo nhãn ground truth
# Output CSV columns:
#   - is_duplicate: {0=unique, 1=duplicate}
#   - modification: {0=original, 1=parser_dup, 2=truncation_dup}
#   - trunc_percentage: {0, 1, 2, 5, 7, 10, 15, 20}
#   - rel_location: {0, 0.25, 0.5, 0.75, 1.0}
```

**Output**: `benchmark_dfs/{tag}.csv`

---

## 📥 Cách tải dữ liệu

### Tùy chọn 1: Clone AdaParse repository

```bash
# Bước 1: Clone repo
git clone https://github.com/allenai/AdaParse.git
cd AdaParse

# Bước 2: Download dữ liệu (tuỳ theo hướng dẫn AdaParse)
# (Thường yêu cầu tải từ cloud storage hoặc request từ tác giả)

# Bước 3: Symlink vào LSHBloomExperiments
cd ../LSHBloomExperiments
ln -s ../AdaParse/joint_to_html .
ln -s ../AdaParse/joint_to_pymupdf .
ln -s ../AdaParse/joint_to_nougat .
ln -s ../AdaParse/joint_to_tesseract .
```

### Tùy chọn 2: Pre-built benchmark

Tác giả LSHBloom có thể cung cấp pre-built benchmark:
- Hãy check **Releases** của repo gốc
- Thường là file `.csv` hoặc `.tar.gz` chứa benchmark CSV

---

## 📊 Thống kê Benchmark

### Mặc định (5.1.4 trong bài báo)

| Thông số | Giá trị |
|---|---|
| Tập tuning | 24,000 documents |
| Tập testing | 50,000 documents |
| Tỷ lệ trùng lặp | 10% → 90% (variable) |
| Cân bằng | 50/50 (parser dup vs truncation dup) |

### Ví dụ output CSV

```
id,text,path,parser,is_duplicate,modification,trunc_percentage,rel_location
1,"The quick...",arxiv/pdf/1.pdf,html,0,0,0,0
2,"The quick...",arxiv/pdf/1.pdf,pymupdf,1,1,0,0
3,"Another doc",arxiv/pdf/2.pdf,html,0,0,0,0
4,"Another",arxiv/pdf/2.pdf,html,1,2,20,0.5
...
```

---

## ⚠️ Lưu ý

1. **Dữ liệu gốc khá lớn** → Yêu cầu storage đáng kể
2. **Quá trình sinh benchmark mất thời gian** → Có thể tối ưu bằng parallelization
3. **Không có pre-built benchmark trong repo** → Cần liên hệ tác giả hoặc sử dụng AdaParse

---

## 📖 Tham khảo

- **LSHBloom paper**: https://arxiv.org/abs/2411.04257
- **AdaParse paper**: Siebenschuh et al., MLSys 2025
- **Repo LSHBloomExperiments**: https://github.com/thangkh02/large-scale-text-dedup

---

## 🧪 Dữ liệu arXiv tự crawl (Benchmark thực tế)

Bên cạnh AdaParse, dự án còn xây dựng bộ dữ liệu riêng bằng cách crawl PDF từ **arXiv** và trích xuất
text theo 3 phương pháp khác nhau. Mỗi phương pháp tạo ra một phiên bản text của cùng tài liệu gốc,
phục vụ xây dựng **near-duplicate benchmark** đa parser.

---

### 1. 📥 Nguồn PDF gốc

PDF được crawl từ **arXiv API** theo nhiều category và query:

| Thông tin | Giá trị |
|---|---|
| Nguồn | arXiv (https://arxiv.org) |
| API | `http://export.arxiv.org/api/query` |
| Category chính | `cs.CL`, `cs.IR`, `cs.LG`, `cs.AI`, `cs.CV`, `cs.SE`, ... |
| Số bài | ~1,579 bài (sau khi dedup theo `arxiv_id`) |
| Thời gian thu thập | 2024–2026 |
| Lưu trữ | `arxiv_ocr_benchmark_workspace/pdfs/*.pdf` |

**Metadata mỗi bài:**

```json
{
  "arxiv_id": "2604.01212v1",
  "title": "YC-Bench: Benchmarking AI Agents...",
  "summary": "...",
  "published": "2026-04-01T17:52:19Z",
  "authors": ["Muyu He", "..."],
  "pdf_url": "https://arxiv.org/pdf/2604.01212v1",
  "primary_category": "cs.CL",
  "categories": ["cs.CL", "cs.AI"]
}
```

File metadata: `arxiv_ocr_benchmark_workspace/metadata/arxiv_search_results_multi_query.jsonl`

---

### 2. 🌐 Data HTML từ arXiv

Text được trích xuất trực tiếp từ **trang HTML full-text** của arXiv (available từ ~2023 với bài dùng LaTeX).

| Thông tin | Giá trị |
|---|---|
| URL pattern | `https://arxiv.org/html/{arxiv_id}` |
| Parser | BeautifulSoup + lxml |
| Coverage | ~90% (1,421/1,579 bài có HTML) |
| Không có HTML | ~10% (bài cũ hoặc không dùng LaTeX) |
| Output | `arxiv_html_reference_workspace/parsed_jsonl/arxiv_html_reference.jsonl` |

**Đặc điểm:**
- HTML được render trực tiếp từ source LaTeX → **text sạch nhất**, ít noise nhất
- Giữ cấu trúc section, paragraph, công thức được thay thế bằng text
- Dùng làm **reference text** chuẩn để so sánh với OCR

**Schema mỗi record:**

```json
{
  "doc_id": "2604.01212v1",
  "parser_name": "arxiv_html",
  "source_type": "html_reference",
  "title": "...",
  "primary_category": "cs.CL",
  "published": "2026-04-01T17:52:19Z",
  "html_url": "https://arxiv.org/html/2604.01212v1",
  "pdf_url": "https://arxiv.org/pdf/2604.01212v1",
  "char_count": 31258,
  "line_count": 842,
  "text_sha1": "a3f1c2...",
  "text": "YC-Bench: Benchmarking AI Agents...",
  "fetch_status": "ok"
}
```

**Thống kê:**

| Chỉ số | Giá trị |
|---|---|
| `fetch_status=ok` | 1,421 bài (90.0%) |
| `fetch_status=failed_404` | 157 bài (9.9%) |
| char_count trung bình | ~50,000 ký tự/bài |
| char_count tối đa | ~645,000 ký tự |

---

### 3. 🔍 Data OCR bằng Tesseract từ PDF

Text được trích xuất bằng **Tesseract OCR** — mỗi trang PDF được render thành ảnh PNG rồi qua OCR engine.

| Thông tin | Giá trị |
|---|---|
| PDF nguồn | Cùng file PDF đã crawl ở mục 1 |
| OCR engine | Tesseract 5.x (`--oem 1 --psm 3`) |
| Render | pdf2image (Poppler backend), DPI=100 |
| Số luồng | 8 workers song song |
| Output | `arxiv_tesseract_ocr_benchmark_workspace/parsed_jsonl/arxiv_tesseract_ocr_all.jsonl` |

**Đặc điểm:**
- Phù hợp cho scan PDF hoặc PDF không có text layer
- Có thể có noise: ký tự nhận dạng sai, lỗi xuống dòng, artifact từ layout
- Cùng `doc_id` với HTML → tạo **positive pair** (cùng bài, khác parser)

**Schema mỗi record:**

```json
{
  "doc_id": "2604.01212v1",
  "source_pdf": "arxiv_ocr_benchmark_workspace/pdfs/2604.01212v1.pdf",
  "parser_name": "tesseract_ocr",
  "text": "YC-Bench Benchm arking AI Agents...",
  "text_sha1": "b7e2a1...",
  "char_count": 24100,
  "page_count": 12,
  "metadata": {
    "title": "...",
    "primary_category": "cs.CL",
    "source": "arxiv"
  }
}
```

---

### 4. 📄 Data parse bằng thư viện Python từ PDF

Text được trích xuất bằng các **thư viện Python text-based** — đọc trực tiếp text layer của PDF, không qua OCR.

| Thông tin | Giá trị |
|---|---|
| PDF nguồn | Cùng file PDF đã crawl ở mục 1 |
| Parser | **PyMuPDF** (`fitz`) và **pypdf** |
| Output (gộp) | `arxiv_ocr_benchmark_workspace/parsed_jsonl/arxiv_multi_parser_ocr_all.jsonl` |
| Output (từng parser) | `arxiv_ocr_benchmark_workspace/parsed_jsonl/pymupdf.jsonl` |
| | `arxiv_ocr_benchmark_workspace/parsed_jsonl/pypdf.jsonl` |

**Đặc điểm:**
- Nhanh hơn OCR nhiều lần, không cần render ảnh
- Chất lượng phụ thuộc vào PDF: tốt với PDF có text layer, kém với scan PDF
- Các parser khác nhau cho text khác nhau → **positive pair** giữa PyMuPDF và pypdf

**Schema mỗi record:**

```json
{
  "doc_id": "2604.01212v1",
  "source_pdf": "arxiv_ocr_benchmark_workspace/pdfs/2604.01212v1.pdf",
  "parser_name": "pymupdf",
  "text": "YC-Bench: Benchmarking AI Agents for Long-Term...",
  "text_sha1": "c9d3f4...",
  "char_count": 29800,
  "page_count": 12,
  "metadata": {
    "title": "...",
    "primary_category": "cs.CL",
    "source": "arxiv"
  }
}
```

---

### 5. 🔗 Quan hệ giữa các nguồn data

Tất cả 3 phương pháp dùng **cùng `doc_id` = `arxiv_id`**, cho phép ghép chéo tạo benchmark:

```
Cùng 1 bài báo arXiv (doc_id = "2604.01212v1")
│
├── parser_name = "arxiv_html"      → HTML text (reference, sạch nhất)
├── parser_name = "tesseract_ocr"   → OCR text  (nhiều noise hơn)
├── parser_name = "pymupdf"         → PDF text-layer (nhanh, ổn định)
└── parser_name = "pypdf"           → PDF text-layer (kết quả khác nhẹ)

→ Mọi cặp (A, B) cùng doc_id nhưng khác parser_name = POSITIVE PAIR (label=1)
→ Cặp (A, B) khác doc_id, cùng primary_category  = HARD NEGATIVE (label=0)
```

**Tổng kết số lượng theo parser:**

| Parser | Số bài | File output |
|---|---|---|
| `arxiv_html` | 1,421 | `arxiv_html_reference.jsonl` |
| `tesseract_ocr` | ~1,578 | `arxiv_tesseract_ocr_all.jsonl` |
| `pymupdf` | ~1,579 | `pymupdf.jsonl` |
| `pypdf` | ~1,579 | `pypdf.jsonl` |
