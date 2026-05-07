# Báo Cáo Lab: CI/CD cho Hệ Thống AI

## 1. Bộ Siêu Tham Số Đã Chọn

Sau khi thực hiện **15 thí nghiệm** với các bộ hyperparameters khác nhau trên MLflow, bộ tham số được chọn là:

```yaml
n_estimators: 200
max_depth: 30
min_samples_split: 2
```

### Lý do lựa chọn

- **`max_depth=30`**: Tăng từ 10 → 30 là thay đổi quan trọng nhất, cho phép cây học các pattern phức tạp hơn trong dữ liệu rượu vang. Các thí nghiệm với max_depth=10 chỉ đạt accuracy ~0.644.
- **`min_samples_split=2`**: Giá trị thấp cho phép cây chia nhỏ nhanh hơn, tăng khả năng fit dữ liệu.
- **`n_estimators=200`**: Đủ lớn để ensemble ổn định mà không tăng thời gian train không cần thiết.

Kết quả: accuracy tăng từ 0.644 → **0.744** (+10 điểm phần trăm), vượt ngưỡng 0.70.

---

## 2. So Sánh Kết Quả Bước 2 và Bước 3

| Chỉ số | Bước 2 (2998 mẫu) | Bước 3 (5996 mẫu) |
|---|---|---|
| accuracy | **0.744** | **0.738** |
| f1_score | 0.743 | 0.737 |
| Số mẫu train | 2998 | 5996 |

**Nhận xét:** Mô hình Bước 3 có accuracy thấp hơn một chút (0.738 vs 0.744) dù train trên gấp đôi dữ liệu. Điều này có thể do dữ liệu mới (train_phase2) chứa nhiều mẫu nhiễu hoặc phân bố khác biệt so với tập ban đầu. Tuy nhiên, accuracy vẫn đạt 0.738 > 0.70, nên mô hình mới vẫn được triển khai.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

### AWS IAM Permissions
- **Vấn đề:** User `ai-lab-user` không có quyền `s3:CreateBucket`, `s3:PutBucketVersioning`, `s3:PutLifecycleConfiguration`.
- **Giải quyết:** Phải nhờ admin cập nhật IAM policy nhiều lần, lần lượt thêm từng permission.

### S3 Bucket Name
- **Vấn đề:** Tên `my-data-bucket` đã bị chiếm toàn cầu. `tuana-my-data-bucket` cũng chưa được cập nhật trong policy.
- **Giải quyết:** Đổi tên bucket thành `tuana-my-data-bucket` và cập nhật IAM policy tương ứng.

### SSH Key Format (Windows)
- **Vấn đề:** File `.ssh/siblimichi.pem` trong project directory bị corruption (có khoảng trắng thừa ở mỗi dòng), khiến SSH báo "invalid format" và "Permission denied".
- **Giải quyết:** Copy lại key từ `~/.ssh/siblimichi.pem` (đúng) sang project directory.

### EC2 Service Won't Start
- **Vấn đề:** `uvicorn` không được cài đặt trên system Python của root user, service liên tục crash và restart spam.
- **Giải quyết:** Cài `fastapi`, `uvicorn`, `pydantic`, `scikit-learn`, `joblib`, `boto3` bằng `sudo pip3 install`.

### Missing S3 Model File
- **Vấn đề:** API server cần model tải từ S3 khi khởi động, nhưng chưa có file trong bucket.
- **Giải quyết:** Chạy train locally và upload model lên `s3://tuana-my-data-bucket/models/latest/model.pkl`.

### GitHub Actions Trigger
- **Vấn đề:** Workflow trigger on `branches: [main]` nhưng repo dùng branch `master`.
- **Giải quyết:** Sửa `branches: [main]` → `branches: [master]` trong `mlops.yml`.

### Hyperparameter Tuning
- **Vấn đề:** Lần chạy đầu tiên accuracy = 0.644 < 0.70, bị chặn bởi eval gate.
- **Giải quyết:** Tăng `max_depth` từ 10 → 30, giảm `min_samples_split` từ 3 → 2. Chạy lại và đạt 0.744.

---

## 4. Kiến Trúc Hệ Thống

```
Code/Data Change → Git Push → GitHub Actions
                                 │
              ┌──────────────────┼──────────────────┐
              ↓                  ↓                  ↓
          Unit Test          Train Job          Deploy Job
          (pytest)          (dvc pull)         (SSH restart)
                           (boto3 upload)
                                 │
                           S3 Bucket
                           (tuana-my-data-bucket)
                                 │
                           EC2 VM
                      (107.20.126.113:8000)
                      (FastAPI + uvicorn)
```

### Các thành phần AWS sử dụng

| Service | Mục đích |
|---|---|
| **EC2 (t3.small)** | VM chạy FastAPI inference server |
| **S3** | Lưu trữ dữ liệu (DVC) và model file |
| **IAM + Instance Profile** | Cấp quyền S3 cho EC2 (không cần key file) |
| **Security Group** | Mở port 22 (SSH) và 8000 (API) |
| **GitHub Actions** | CI/CD pipeline (test → train → eval → deploy) |

---

## 5. Kết Luận

Hệ thống MLOps đã hoạt động hoàn chỉnh:
- **Bước 1**: Thực nghiệm cục bộ với MLflow → chọn bộ hyperparameters tối ưu.
- **Bước 2**: CI/CD tự động từ push code → triển khai lên EC2.
- **Bước 3**: Huấn luyện liên tục khi có dữ liệu mới (dvc push → git push → pipeline tự động).

Mô hình tốt nhất đạt accuracy **0.744** trên 2998 mẫu. Khi thêm 2998 mẫu mới, accuracy giảm nhẹ xuống **0.738** nhưng vẫn vượt ngưỡng deploy.
