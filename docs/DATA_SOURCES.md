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
