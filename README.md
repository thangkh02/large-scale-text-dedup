# LSHBloom: Khử Trùng Lặp Văn Bản Quy Mô Internet

> Dựa trên bài báo: *"LSHBloom: Internet-Scale Text Deduplication"* — Arham Khan et al., PVLDB 2025.

Kho lưu trữ này chứa các thực nghiệm và benchmark để tái sản xuất các chiến lược khử trùng lặp văn bản, bao gồm **LSHBloom**, **MinHashLSH**, **CCNet**, **DCLM** và **Dolma**. Ngoài ra, kho cũng bao gồm một bộ sinh dữ liệu benchmark tổng hợp để đánh giá các chiến lược này trong điều kiện có kiểm soát.

---

## Giới thiệu

Các cơ sở dữ liệu văn bản hiện đại ở quy mô internet yêu cầu quy trình khử trùng lặp mạnh mẽ nhằm duy trì tính toàn vẹn dữ liệu và giảm chi phí xử lý. **MinHashLSH** là phương pháp phổ biến nhất cho bài toán phát hiện gần-trùng lặp (near-duplicate detection), được sử dụng rộng rãi trong các pipeline tiền huấn luyện LLM (GPT-3, Llama 3, Gopher, OLMo, RedPajama...). Tuy nhiên, chỉ số MinHashLSH truyền thống dựa trên cây (tree) hoặc hashmap có **dung lượng đĩa rất lớn** và **độ trễ cao** do truy cập bộ nhớ không liên tục.

**LSHBloom** là phương pháp mới thay thế cấu trúc chỉ mục truyền thống bằng một mảng các **Bloom filter độc lập**, giúp:

- **Tăng thông lượng 12×** so với MinHashLSH tiêu chuẩn trên 39 triệu tài liệu (tập peS2o).
- **Giảm dung lượng đĩa 18×** trong khi vẫn đạt recall và precision gần như tương đương.
- Cho phép khử trùng lặp **hàng tỷ bản ghi** trên phần cứng phổ thông.

---

## Cài đặt

### Yêu cầu hệ thống

Đảm bảo đã cài **Python 3.10** và **Rust** (để biên dịch `pyhash-archive`).

### Các bước cài đặt

1.  **Cài đặt các gói phụ thuộc chung**:
    ```bash
    pip install -r requirements.txt
    ```

2.  **Cài đặt các thư viện nhúng**:
    Kho lưu trữ này chứa các phiên bản đã được chỉnh sửa của một số thư viện trong thư mục `dedup/`. Cần cài đặt chúng thủ công:

    *   **CCNet**:
        Làm theo hướng dẫn trong repository để build CCNet, sau đó:
        ```bash
        pip install dedup/cc_net
        ```

    *   **Datasketch**:
        ```bash
        pip install dedup/lsh/datasketch
        ```

    *   **PyHash Archive** (Yêu cầu Rust/Cargo):
        ```bash
        cd dedup/lsh/pyhash-archive
        pip install .
        ```

---

## Cấu trúc thư mục

### `dedup/`
Chứa các cài đặt của các chiến lược khử trùng lặp và harness để chạy thực nghiệm.

*   **`dedup_parsing_harness.py`**: Định nghĩa lớp trừu tượng `DedupHarness`. Tất cả các thực nghiệm kế thừa lớp này để đảm bảo quy trình nhất quán (đọc dữ liệu → khử trùng lặp → tính điểm). Hàm `score()` tính các metric: precision, recall, F1, AUC-ROC, balanced accuracy.
*   **`cc_net/`**: Mã nguồn thư viện CCNet (Facebook Research).
*   **`ccnet/`**: Chứa `ccnet.py` — script chạy thực nghiệm CCNet (hash SHA1 cấp đoạn văn).
*   **`dclm/`**: Chứa `dclm.py` — script chạy thực nghiệm DCLM dùng Bloom filter theo n-gram (tokenizer UniSeg).
*   **`dolma/`**: Chứa `dolma.py` — script chạy thực nghiệm Dolma (cấp đoạn văn hoặc n-gram, tokenizer grapheme cluster Unicode).
*   **`lsh/`**: Các cài đặt LSH (Locality Sensitive Hashing).
    *   `datasketch/`: Thư viện `datasketch` nhúng (MinHash, MinHashLSH, MinHashLSHBloom).
    *   `pyhash-archive/`: Thư viện `pyhash-archive` nhúng — hàm hash tốc độ cao viết bằng Rust (tối ưu hóa vector số nguyên 128-bit, nhanh hơn Python 94%).
    *   `lsh.py`: Runner cho MinHashLSH tiêu chuẩn (backend Redis).
    *   `lsh_bloom.py`: Runner cho **LSHBloom** — thay thế Redis bằng mảng Bloom filter.
    *   `minhash.py`: Tiện ích tính MinHash song song (32 worker) cho batch JSONL.
*   **`writers.py`**: Tiện ích ghi kết quả ra CSV.

### `synthetic_benchmark/`
Các script sinh và quản lý tập dữ liệu benchmark tổng hợp để đánh giá độ trung thực khử trùng lặp.

*   **`gen_dedup_benchmark.py`**: Script chính để sinh tập benchmark. Tạo dữ liệu trùng lặp bằng cách lấy mẫu từ nhiều parser khác nhau (HTML, PyMuPDF, Nougat, Tesseract) và tạo các phiên bản được cắt ngẫu nhiên (`truncation`).
*   **`config.py`**: Các hằng số cấu hình (đường dẫn, kích thước dữ liệu `DATA_SIZE=50_000`...).
*   **`estimate_para.py` / `estimate_ngram.py`**: Ước tính số lượng đoạn văn / n-gram trong corpus — cần thiết để khởi tạo Bloom filter đúng kích thước cho DCLM và Dolma.
*   **`dedup_benchmark_utils.py`**: Các hàm hỗ trợ cho bộ sinh benchmark.
*   **`make_jsonl.py`**: Chuyển đổi file CSV benchmark sang định dạng JSONL để chạy khử trùng lặp.

---

## Phương pháp luận benchmark

Theo bài báo (Mục 5.1.4), tập benchmark được xây dựng như sau:

- **Nguồn dữ liệu**: Các bài báo khoa học từ nhiều lĩnh vực (toán, vật lý, hóa học, sinh học, kinh tế, kỹ thuật, y học), mỗi bài có hai phiên bản (HTML và PDF được parse bởi PyMuPDF / Nougat / Tesseract).
- **Loại trùng lặp được mô phỏng**:
  - **`modification=1` (parser duplicate)**: Cùng tài liệu gốc nhưng được parse bởi các công cụ khác nhau → gần-trùng lặp do nhiễu parsing.
  - **`modification=2` (truncation duplicate)**: Phiên bản bị cắt xén ngẫu nhiên (tối đa 20% nội dung) → mô phỏng lỗi parse OCR hoặc nội dung bị rút gọn.
- **Tập tuning**: 24.000 tài liệu, cân bằng giữa hai loại trùng lặp.
- **Tập testing**: 50.000 tài liệu, với tỷ lệ trùng lặp từ 10% đến 90%.

---

## Tham số tốt nhất (theo bài báo)

| Thuật toán | Kích thước N-gram | Ngưỡng tương đồng |
|---|---|---|
| **MinHashLSH** | — | 0.5 (Jaccard) |
| **LSHBloom** | — | 0.5 (Jaccard) |
| Dolma-Ngram | 5 | 0.2 |
| DCLM | 5 | 0.2 |
| Dolma | — | 0.2 |
| CCNet | — | 0.2 |

> MinHashLSH và LSHBloom đạt F1 score cao nhất (~0.84) với 256 permutations và threshold=0.5.

---

## Chạy thực nghiệm

### Bước 1 — Sinh dữ liệu benchmark

```bash
python synthetic_benchmark/gen_dedup_benchmark.py \
    --N-per-source 3000 \
    -p 0.5 \
    -o my_benchmark
```

### Bước 2 — Chuyển đổi sang JSONL

```bash
python synthetic_benchmark/make_jsonl.py
```

### Bước 3 — Ước tính số lượng (chỉ cần cho DCLM / Dolma)

```bash
python synthetic_benchmark/estimate_ngram.py
python synthetic_benchmark/estimate_para.py
```

### Bước 4 — Chạy khử trùng lặp

Thay `<benchmark_tag>` bằng tag của tập dữ liệu (ví dụ: `50k`).

*   **CCNet** (hash SHA1 cấp đoạn văn):
    ```bash
    python dedup/ccnet/ccnet.py --input <benchmark_tag> --sim-threshold 0.2
    ```

*   **DCLM** (Bloom filter n-gram, tokenizer UniSeg):
    ```bash
    python dedup/dclm/dclm.py --input <benchmark_tag> --sim-threshold 0.2 --ngram-size 5
    ```

*   **Dolma**:
    ```bash
    # Chế độ đoạn văn (yêu cầu estimate_para.py)
    python dedup/dolma/dolma.py --input <benchmark_tag> --sim-threshold 0.2

    # Chế độ n-gram (yêu cầu estimate_ngram.py)
    python dedup/dolma/dolma.py --input <benchmark_tag> --sim-threshold 0.2 --ngram-mode --ngram-size 5
    ```

*   **MinHashLSH** (yêu cầu Redis đang chạy trên port 6379):
    ```bash
    python dedup/lsh/lsh.py \
        --input <benchmark_tag> \
        --sim-threshold 0.5 \
        --num-perm 256 \
        --redis-port 6379 \
        --force-compute-minhash
    ```

*   **LSHBloom** ⭐ (không cần Redis, dùng Bloom filter):
    ```bash
    python dedup/lsh/lsh_bloom.py \
        --input <benchmark_tag> \
        --sim-threshold 0.5 \
        --num-perm 256 \
        --force-compute-minhash
    ```

    Để thực nghiệm với false positive rate tùy chỉnh:
    ```bash
    python dedup/lsh/lsh_bloom.py \
        --input <benchmark_tag> \
        --sim-threshold 0.5 \
        --num-perm 256 \
        --fp 1e-5
    ```

---

## Kết quả thực nghiệm (tóm tắt từ bài báo)

### Độ trung thực (50.000 tài liệu, tỷ lệ trùng lặp 10%–90%)

| Thuật toán | Precision | Recall | F1 |
|---|---|---|---|
| **LSHBloom** | ~Cao nhất | 0.7–0.9 | **~0.84** |
| **MinHashLSH** | ~Cao nhất | 0.7–0.9 | **~0.85** |
| DCLM | Trung bình | ~Cao | ~0.81 |
| Dolma-Ngram | Thấp | ~Cao | ~0.80 |
| Dolma | Trung bình | <0.4 | ~0.55 |
| CCNet | Thấp | Thấp | ~0.55 |

### Hiệu suất quy mô (39 triệu tài liệu — tập peS2o)

| Thuật toán | Thời gian | Dung lượng đĩa |
|---|---|---|
| MinHashLSH | ~37 giờ | >200 GB |
| **LSHBloom** | **~3 giờ (nhanh hơn 12×)** | **~11 GB (nhỏ hơn 18×)** |
| CCNet | ~Nhanh | ~Nhỏ |
| Dolma | ~Nhanh | ~Nhỏ |
| DCLM / Dolma-Ngram | **>200 giờ** (không khả thi) | — |

### Ước tính cho 5 tỷ tài liệu

| Thuật toán | Thời gian ước tính | Dung lượng chỉ mục |
|---|---|---|
| MinHashLSH | ~200 ngày | ~277 TB |
| **LSHBloom** ($p_{eff}=10^{-5}$) | **~15 ngày** | **~8.3 TB** |

---

## Phân tích lỗi của LSHBloom

Tỷ lệ lỗi của LSHBloom có thể được giới hạn phân tích như sau:

$$FP_{bloom} = FP_{lsh} + (1 - FP_{lsh}) \cdot (p_{effective} + b/N)$$

$$FN_{bloom} = (1 - (p_{effective} + b/N)) \cdot FN_{lsh}$$

Trong đó $p_{effective} = 1 - (1-p)^b$ với $b$ là số band và $p$ là FP rate của mỗi Bloom filter riêng lẻ.

---

## Trích dẫn

```bibtex
@article{khan2025lshbloom,
  title     = {LSHBloom: Internet-Scale Text Deduplication},
  author    = {Khan, Arham and Underwood, Robert and Siebenschuh, Carlo and
               Babuji, Yadu and Ajith, Aswathy and Hippe, Kyle and
               Gokdemir, Ozan and Brace, Alexander and Chard, Kyle and Foster, Ian},
  journal   = {Proceedings of the VLDB Endowment},
  volume    = {19},
  number    = {1},
  year      = {2025},
  doi       = {XX.XX/XXX.XX}
}
```
