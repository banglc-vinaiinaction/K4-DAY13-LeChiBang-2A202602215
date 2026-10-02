# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: K4-DAY13-LeChiBang
- Thành viên: xem `TEAMMATES.md` (Lê Chí Bằng - MSSV: 2A202602215).
- Trạng thái: `executed-by-individual` (Chạy trực tiếp trên máy cá nhân).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lê Chí Bằng; 2026-10-02 19:08:27 UTC+7; Fedora Linux 44 (x86_64 / amd64), CPU Intel Core i5-6500T @ 2.50GHz, 24 GB RAM.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64`; Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; commit: `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` (chuyển đổi từ KITTI demo 000008, 17.238 điểm finite, CC BY-NC-SA 3.0); SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`; SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window `[0, -19.84, -2.5, 47.36, 19.84, 0.5]`; score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ tư là constant placeholder (RGB=0, reflectance thật bị tước bỏ theo thỏa thuận adaptation); `z_ground` được ước lượng từ thuật toán phân tách mặt đất cục bộ: `z_ground = 0.075 m`.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/summary.csv`, `boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png` | Chỉ phát hiện duy nhất 1 xe (`vehicles`, score 0.322) tại x≈13.15m. Do thiếu bù cao độ cảm biến (delta=0), hầu hết các đối tượng nằm lệch ngoài phạm vi anchor z mà mô hình được huấn luyện, dẫn tới bỏ sót gần như toàn bộ xe. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/summary.csv`, `boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png` | Baseline chuẩn: Phát hiện 13 hộp (10 vehicles, 1 two-wheels, 2 pedestrian) trải đều từ x≈3.7m đến x≈55.6m. Các xe gần đạt confidence cao (0.80 - 0.93). Cao độ tâm hộp mean_z = 1.034m khớp với hình học xe thực tế trên mặt đất. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/summary.csv`, `boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png` | Toàn bộ 6 hộp dự đoán đều bị gán nhãn `pedestrian`, không có nhãn `vehicles` nào vượt qua ngưỡng 0.3. Khi kích thước pillar tăng gấp đôi (32cm) mà không retrain model, đặc trưng hình học của vật thể lớn bị méo mó, dẫn đến phân loại nhầm nghiêm trọng. |

- **A/B — chỉ đổi delta**: A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-0-voxel-0.16.png` cho thấy khi delta=0, đám mây điểm nằm quá thấp so với anchor mong đợi của PointPillars, khiến bộ phát hiện bỏ sót 12 đối tượng. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là liệu với cảm biến đặt tại các vị trí lắp đặt khác nhau (như nóc xe vs cản trước), thuật toán ước lượng ground tự động có đủ bù trừ sai lệch nếu thiếu thông số extrinsic chính xác hay không.
- **B/C — chỉ đổi pillar**: B có 13 hộp; C có 6 hộp. Ảnh `side-demo-delta-1.73-voxel-0.32.png` cho thấy cấu trúc hộp thay đổi hoàn toàn: mất toàn bộ xe lớn, xuất hiện 6 hộp pedestrian phân tán. Không có đủ bằng chứng để kết luận C tốt hơn; thực tế C cho kết quả tệ hơn vì model checkpoint dùng cố định được tối ưu ở độ phân giải lưới 0.16m.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào**: Góc Side (hình chiếu x-z) rất hữu ích để kiểm tra cao độ đáy hộp tiếp xúc mặt đường và chiều cao vật thể, nhưng làm triệt tiêu trục y và góc xoay hướng di chuyển (yaw). Vì vậy nhìn một mình góc Side không thể phân biệt xe đi xuôi hay ngược chiều và dễ nhìn nhầm các vật thể chồng lấn theo chiều ngang.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**: Không được import bất kỳ JSON nào trong các kết quả demo này vào CVAT nguồn của Robotaxi. Cả ba file đều chạy trên KITTI demo, không phản ánh hệ tọa độ, cảm biến và bài gán nhãn của Robotaxi thật. Khi làm việc với job nguồn Robotaxi, chỉ nạp pre-label tương ứng đúng frame được cấp phép từ portal.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm từng hộp | `case-correct.json`, `side-correct.png`: giữ nguyên bản dự đoán gốc từ lượt B, các hộp đều đặt đáy sát mặt đất ước lượng. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi | **Dừng batch (Dừng pipeline)** | `case-batch-z.json`, `side-batch-z.png`: Toàn bộ 13/13 hộp đồng loạt bị tụt sâu xuống lòng đất 1.805m (bằng đúng delta + z_ground). Đây là lỗi pipeline do thiếu bước biến đổi tọa độ nghịch đảo, phải báo engineering/data ops xử lý hệ thống chứ không sửa tay từng hộp. |
| case-one-box-z | 1 / 13 | -1.805 m | Không đổi | **Kiểm từng hộp (Sửa cục bộ)** | `case-one-box-z.json`, `side-one-box-z.png`: Duy nhất hộp đầu tiên bị lệch cao độ z, 12 hộp còn lại vẫn đúng vị trí mặt đường. Đây là lỗi cục bộ của một đối tượng cụ thể, người gán nhãn cần dùng các góc nhìn để chỉnh đáy lại cho đúng. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Thành viên: Lê Chí Bằng (MSSV: 2A202602215)
- **Vai trò đã thực hiện**: Thiết lập môi trường Docker/Podman trên hệ thống Fedora Linux x86_64; cấu hình cơ chế tương thích SELinux cho container; trực tiếp vận hành script `student-bundle.py` chạy đủ ba lượt A/B/C và bộ ca QC; trích xuất số liệu `summary.csv` và phân tích đối chiếu các file kết quả.
- **Quan sát A/B/C có dẫn chứng file**:
  - Khi so sánh `run-A` với `run-B`, số hộp tăng từ 1 lên 13. Trong `run-A/boxes-demo-delta-0-voxel-0.16.json`, chiếc xe duy nhất nhận được ở tọa độ `z=0.330m` với score thấp (`0.322`), trong khi ở `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, xe tương tự tại `x=14.76m` có `z=0.899m` và score đạt `0.928`. Điều này chứng minh việc bù delta=1.73m đưa các điểm LiDAR về đúng dải cao độ mà PointPillars học được.
  - Khi so sánh `run-B` với `run-C`, kích thước pillar tăng từ 0.16m lên 0.32m khiến toàn bộ nhãn `vehicles` biến mất và chỉ còn 6 nhãn `pedestrian` trong `run-C/boxes-demo-delta-1.73-voxel-0.32.json`. Việc gom điểm thô gấp đôi làm mất cấu trúc cụm điểm hình hộp chữ nhật của ô tô.
- **Diễn giải phép biến đổi z thuận/nghịch**:
  - *Phép thuận trước khi vào model*: $z_{\text{in}} = z_{\text{raw}} - z_{\text{ground}} + \delta$. Điểm được đưa về mốc mặt đất cục bộ rồi nâng theo độ cao giả định của cảm biến để khớp với anchor pretrained.
  - *Phép nghịch sau khi model dự đoán*: $z_{\text{box, source}} = z_{\text{box, model}} + z_{\text{ground}} - \delta$. Bắt buộc phải thực hiện bước này để trả tâm và đáy hộp về đúng hệ tọa độ PCD gốc. Nếu bỏ quên bước này, toàn bộ batch hộp sẽ bị lệch đúng một khoảng $-(z_{\text{ground}} + \delta) = -1.805\text{ m}$.
- **Quyết định khi gặp lỗi batch vs lỗi một hộp**:
  - *Lỗi batch* (như `case-batch-z`): Phát hiện toàn bộ hộp cùng bị tụt hoặc bay lên so với mặt đường -> Dừng gán nhãn ngay lập tức, thông báo quản trị kiểm tra lại pipeline tiền xử lý và ma trận extrinsic; tuyệt đối không sửa tay từng hộp vì sẽ tạo ra nhãn sai lệch và tốn công vô ích.
  - *Lỗi một hộp* (như `case-one-box-z`): Do vùng bị che khuất (occlusion) hoặc mật độ điểm quá thưa ở xa -> Sử dụng chế độ xem đa góc (Top, Front, Side) trong CVAT/3D viewer và đối chiếu ảnh camera để ước lượng đáy cục bộ và điều chỉnh kích thước hộp.
- **Phát hiện lỗi Pipeline thực tế trên 30 jobs Robotaxi**:
  - Khi áp dụng kiểm tra hàng loạt tọa độ và kích thước của hơn 1000 hộp qua CVAT API trên 30 jobs thực tế được giao, em đã phát hiện ra 2 lỗi hệ thống (batch error) nghiêm trọng từ pipeline tạo pre-label:
    1. **Lỗi gán ngược nhãn 100%**: Kích thước trung bình của nhãn `pedestrian` lên tới 1.66m x 4.05m (kích thước ô tô), trong khi nhãn `vehicles` lại là 0.68m x 0.74m (kích thước người).
    2. **Lỗi Transform Z**: Tọa độ Bottom-Z trung bình của tất cả các hộp nằm ở mức 0.0m. Trong hệ tọa độ LiDAR, mặt đất thường nằm ở khoảng -1.73m. Suy ra toàn bộ các hộp đang treo lơ lửng cách mặt đất đúng một lượng delta.
  - Dựa vào nguyên tắc xử lý lỗi batch trong bài, thay vì ngồi chỉnh tay trục Z cho hàng ngàn hộp (sẽ làm hỏng dữ liệu), em đã quyết định dừng thao tác sửa trục Z và chỉ hoàn thiện 15 jobs theo yêu cầu giảm tải, đồng thời ghi nhận bằng chứng này để thông báo cho LC kiểm tra lại transform pipeline.
- **Điều còn chưa chắc chắn**: Do file `demo.pcd` là bản KITTI đã qua chuyển đổi loại bỏ reflectance thật (đặt RGB=0), em chưa đánh giá được mức độ suy giảm chất lượng nhận diện của mô hình đối với các vật thể có độ phản xạ thấp (như lốp xe, mặt đường ướt) so với khi có kênh intensity chuẩn.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
