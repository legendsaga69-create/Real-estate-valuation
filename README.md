# Hệ thống Phân tích Thị trường & Dự đoán Giá Bất động sản

Dự đoán đơn giá (VNĐ/m²) phục vụ thẩm định bất động sản tại các khu vực thiếu dữ liệu tham chiếu — đánh giá theo đúng chuẩn độ chính xác thẩm định thực tế, không chỉ dựa vào độ khớp thống kê.

## Bối cảnh nghiệp vụ

Không phải khu vực nào cũng có đủ dữ liệu giao dịch tham chiếu để xác lập giá trị thị trường đáng tin cậy. Ở những khu vực hạn chế dữ liệu, việc định giá vẫn phụ thuộc nhiều vào kinh nghiệm và nhận định cá nhân của thẩm định viên — cách làm này thiếu cơ sở dựa trên dữ liệu, và làm tăng rủi ro sai lệch giá.

Chuẩn mực thẩm định giá Việt Nam đặt ra một ngưỡng khắt khe hơn nhiều so với chỉ dựa vào độ khớp thống kê: ước tính phải nằm trong biên độ **±10% so với giá trị thị trường** mới được coi là đạt. Project này đánh giá mọi model theo đúng ngưỡng thực tế đó, không chỉ dừng ở R².

## Mục tiêu

Xây dựng pipeline dự đoán đơn giá cho các khu vực thiếu dữ liệu — giảm tải khối lượng công việc thủ công cho trợ lý thẩm định viên — và kiểm chứng kết quả theo đúng chuẩn độ chính xác thẩm định thực tế (PE10, COD, PRD) thay vì chỉ dựa vào độ khớp thống kê.

## Dữ liệu

- 2.706 hồ sơ thẩm định trên nhiều tỉnh thành
- Các trường gốc: loại đất, loại nhà, địa chỉ hành chính (phường/huyện/tỉnh), kích thước diện tích (tổng, pháp lý, chiều rộng, chiều dài), tỷ lệ đất ở, đơn giá do thẩm định viên ước tính
- Target: `unitpriceEstimatedByStaff` (VNĐ/m²), log-transform trước khi đưa vào model

## Phương pháp

### 1. EDA có hệ thống (trên dữ liệu gốc, trước khi xử lý)
- Biến liên tục: phân phối, độ lệch (skew), tương quan Pearson/Spearman (cả bản gốc và log-scale)
- Biến phân loại: ANOVA / Kruskal-Wallis + Eta-squared (tương đương Chi-Square/Cramér's V nhưng dùng cho target liên tục)
- **Sửa lỗi tách địa chỉ:** trường địa chỉ gốc lẫn lộn cách gọi hành chính cũ và mới (sau đợt sáp nhập hành chính 2025) trong ngoặc đơn (vd. "... Thị xã Sóc Trăng (nay là thành phố Sóc Trăng), Tỉnh Sóc Trăng"); regex ban đầu lấy nguyên khối trong ngoặc, bỏ sót các cấp không được nhắc tới trong đó (như tỉnh) — đã viết lại để ghép từng cấp hành chính độc lập, ưu tiên phần "nay là" nhưng fallback về địa chỉ gốc nếu thiếu, cộng thêm fallback theo vị trí cho các địa chỉ 2 cấp mới không có tiền tố "Tỉnh/Thành phố"

### 2. Pipeline chống rò rỉ dữ liệu (leakage-safe)
- Tách train/test trước mọi bước xử lý
- Phân cụm diện tích bằng GMM (đất nhỏ/lớn), chỉ fit trên train
- Target encoding vị trí theo phương pháp out-of-fold (OOF) (`ward`, `loc_segment = ward+province+segment`) — mỗi dòng được encode chỉ bằng thông tin từ 4 fold còn lại, không bao giờ dùng target của chính nó
- Dùng chung **1 bộ `StratifiedKFold`** xuyên suốt cho mọi bước encoding lẫn đánh giá model, đảm bảo các fold luôn khớp nhau, tránh rò rỉ chéo tinh vi giữa các fold

### 3. Đánh giá theo chuẩn thẩm định (không chỉ R²)
Bên cạnh các metric hồi quy thông thường (R², MAPE, RMSE, MAE), bổ sung các chỉ số chuẩn mass-appraisal phản ánh trực tiếp ngưỡng ±10%:
- **PE10** — % ước tính nằm trong biên độ ±10% so với giá trị thực
- **COD** (Coefficient of Dispersion) — độ phân tán sai số quanh tỷ lệ trung vị
- **PRD** (Price-Related Differential) — độ thiên lệch định giá giữa BĐS giá trị thấp và cao

### 4. Feature Engineering — kiểm chứng bằng ablation, không giả định
Các feature theo nghiệp vụ thẩm định thực tế (mô phỏng Phương pháp So sánh trực tiếp — 3 BĐS so sánh, điều chỉnh theo đặc điểm riêng) được thử nghiệm và đánh giá có kiểm soát:
- Định giá theo K-BĐS so sánh gần nhất (mô phỏng định giá 3-BĐS so sánh) — thử nghiệm từ K=3 đến K=100; kết quả không ổn định, chưa bao giờ vượt qua baseline
- Xác định nguyên nhân gốc qua phân tích cỡ nhóm: phần lớn nhóm `loc_segment` có **dưới 15 hồ sơ so sánh**, quá thưa để feature dạng so sánh khái quát hóa tốt

### 5. So sánh model
So sánh 13 model hồi quy qua 5-fold CV: Linear/Ridge/Lasso/ElasticNet, Decision Tree, Random Forest/Extra Trees/Bagging, Gradient Boosting/XGBoost/LightGBM/CatBoost/AdaBoost — đánh giá cả trên out-of-fold lẫn tập test độc lập.

## Kết quả

| Model | CV R² | Test R² | Test MAPE | Test PE10 | Test COD | Test PRD |
|---|---|---|---|---|---|---|
| LightGBM | 0.886 | 0.883 | 53.4% | 16.1% | 52.2 | 1.36 |
| XGBoost | 0.886 | 0.883 | 52.4% | 20.7% | 49.8 | 1.34 |
| CatBoost | 0.883 | 0.883 | 52.3% | 17.3% | 51.9 | 1.40 |
| Extra Trees | 0.885 | 0.884 | 49.8% | 23.6% | 48.8 | 1.32 |
| Gradient Boosting | 0.886 | 0.883 | 53.6% | 17.7% | 51.7 | 1.36 |

*(Bảng so sánh đầy đủ 13 model, bao gồm các model tuyến tính, trong notebook.)*

## Phát hiện chính

**R² ~0.88 trông rất ấn tượng — nhưng chỉ 14–24% ước tính nằm trong biên độ ±10% theo chuẩn thẩm định (PE10).** Đây là phát hiện trung tâm của project: R² cao chủ yếu đến từ việc lấy trung bình theo nhóm vị trí (giải thích tốt phần biến thiên giá *giữa* các khu vực), trong khi phần biến thiên *trong cùng 1 khu vực* — phần quyết định 1 ước tính cụ thể có đạt chuẩn thẩm định hay không — vẫn phần lớn chưa được giải thích.

Phân tích cỡ nhóm đã xác định trần hiệu năng này đến từ:
1. **Nhóm so sánh quá thưa** — phần lớn khu vực có quá ít hồ sơ lịch sử để các feature dạng so sánh ổn định được.
2. **Thiếu thuộc tính cấp tài sản** — dataset thiếu độ rộng đường, tách biệt khỏi diện tích lô đất, hướng nhà, tình trạng pháp lý chi tiết, và các đặc điểm khác mà thẩm định viên dùng để điều chỉnh trong biên độ ±10%.

Đây là giới hạn của dữ liệu, không phải giới hạn của mô hình — được xác nhận qua việc thử nhiều thuật toán, nhiều feature được thiết kế theo nghiệp vụ, tất cả đều hội tụ về cùng 1 trần hiệu năng.

## Công cụ sử dụng

Python · pandas · scikit-learn · LightGBM · CatBoost · XGBoost · Optuna · scipy.stats · GaussianMixture · matplotlib · seaborn

## Hướng phát triển tiếp theo

- Thu thập thêm thuộc tính cấp tài sản (độ rộng đường, tình trạng mặt bằng, môi trường tự nhiên,...) để thu hẹp khoảng cách PE10
- Xây dựng module chọn BĐS so sánh + điều chỉnh bám sát hơn Phương pháp So sánh trực tiếp, khi cỡ nhóm đủ lớn
- Theo dõi biến động PE10/COD/PRD nếu đưa vào triển khai thực tế, song song với theo dõi hiệu năng model thông thường
