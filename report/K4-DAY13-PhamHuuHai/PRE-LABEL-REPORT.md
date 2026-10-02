# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân (theo hướng dẫn của thầy, làm cá nhân không làm nhóm)
- Thành viên: xem `TEAMMATES.md` — Phạm Hữu Hải / 2A202602098, tự thực hiện tất cả vai trò.
- Trạng thái: `executed-by-group`
- Người thực sự chạy: Phạm Hữu Hải; ngày/giờ: 2026-10-02 ~09:31–09:33 UTC; hệ máy: Linux x86_64 (amd64)
- Image tag: `day13-pointpillars:lc-20261001-amd64`; image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD được cấp: `demo.pcd` (KITTI Student); frame_id: `demo`; input SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI pretrained `epoch_160.pth`; SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity: reflectance bị bỏ trong bản PCD, dùng RGB=0 làm placeholder (adapter kênh hằng); z_ground ước lượng từ dữ liệu = 0.075 m

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | boxes-demo-delta-0-voxel-0.16.json / side-demo-delta-0-voxel-0.16.png / summary.csv | Chỉ 1 hộp vehicles ở x≈13.2m, z≈0.33m. Ảnh Side cho thấy hộp nằm sát mặt đường, gần như không nhận ra đối tượng khác. |
| B | 1.73 | 0.16 | 13 | 1.034 | boxes-demo-delta-1.73-voxel-0.16.json / side-demo-delta-1.73-voxel-0.16.png / summary.csv | 13 hộp gồm vehicles=10, pedestrian=2, two-wheels=1. Hộp trải từ x≈3.7m đến x≈55.6m. Ảnh Side cho thấy hộp bám cụm điểm rõ ràng hơn. |
| C | 1.73 | 0.32 | 6 | 1.091 | boxes-demo-delta-1.73-voxel-0.32.json / side-demo-delta-1.73-voxel-0.32.png / summary.csv | 6 hộp toàn pedestrian, mất hết vehicles và two-wheels. Hộp tập trung vùng x≈9–34m, ít và nhỏ hơn B. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Khi delta=0, model gần như không nhận ra đối tượng vì input chưa được dịch về đúng giả định cao độ sensor của checkpoint KITTI (~1.73m). Khi delta=1.73, model nhận ra 13 đối tượng thuộc 3 class. Ảnh Side run-A chỉ có 1 hộp nhỏ ở x≈13m, trong khi Side run-B có hộp rải khắp scene từ x≈3m đến x≈56m. Đây là chạy lại model trên input đã dịch z, không chỉ dịch hộp cũ — nên số hộp thay đổi hoàn toàn (1→13), không phải mọi hộp lệch đúng 1.73m. Điều em còn chưa chắc: không rõ hộp duy nhất trong A có tương ứng với hộp nào trong B hay không, vì model chạy lại hoàn toàn.
- B/C — chỉ đổi pillar: B có 13 hộp (vehicles=10, pedestrian=2, two-wheels=1); C có 6 hộp (pedestrian=6). Pillar lớn gấp đôi (0.16→0.32) làm mất toàn bộ class vehicles và two-wheels, chỉ còn pedestrian. Ảnh Side run-C cho thấy hộp ít hơn, tập trung khu vực gần (x≈9–34m), không còn hộp xa (x≈40–56m). Checkpoint được train cho pillar 0.16m; dùng 0.32m thay đổi biểu diễn đầu vào khiến model không nhận đúng xe. Có đủ bằng chứng để kết luận pillar ảnh hưởng lớn đến kết quả? Có — class thay đổi hoàn toàn, nhưng C không train riêng cho pillar lớn nên chưa thể nói pillar 0.32 luôn kém; chỉ kết luận checkpoint này không phù hợp pillar 0.32.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw: Side là hình chiếu x-z, chồng y, nên không phân biệt được đối tượng cùng x nhưng khác y. Yaw không đọc được từ Side. ROI front-window bỏ vùng phía sau, nên vật phía sau không phải miss của model.
- JSON nào còn chưa đủ cơ sở để import? Cả ba lượt A/B/C đều là thí nghiệm trên KITTI demo, không phải Robotaxi, không import vào CVAT. Cần kiểm tiếp: đối chiếu prediction Robotaxi thật khi nạp pre-label trên portal.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | 0 m | Không — giữ nguyên B gốc | Baseline, không cần hành động | Ảnh Side giống hệt run-B; hộp bám cụm điểm |
| case-batch-z | 13/13 | −1.805 m (z_ground+delta = 0.075+1.73) | Không — chỉ z thay đổi, class/x/y/yaw giữ nguyên | **Dừng batch** — báo LC kiểm pipeline vì toàn bộ hộp bị lệch cùng lượng | Ảnh Side: tất cả 13 hộp chìm dưới z=0 (khoảng z≈−0.8 đến −1.8m), xa khỏi cụm điểm hoàn toàn. Lỗi hệ thống, không phải lỗi từng đối tượng |
| case-one-box-z | 1/13 | −1.805 m cho 1 hộp | Không — chỉ z của hộp đầu tiên thay đổi | **Kiểm từng hộp** — 12 hộp bình thường, 1 hộp lệch riêng lẻ cần kiểm bằng nhiều view | Ảnh Side: 12 hộp vẫn bám cụm điểm, 1 hộp (x≈8m) chìm xuống z≈−1m. Đây là lỗi đối tượng đơn lẻ, không phải lỗi pipeline |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Phạm Hữu Hải — 2A202602098

- **Vai trò đã làm:** Tự thực hiện toàn bộ (vận hành lệnh, kiểm cấu hình JSON, xem hình học Side, ghi log) do làm cá nhân theo hướng dẫn của thầy.
- **Một quan sát A/B/C có dẫn file hoặc hộp/vùng:** So A/B trong ảnh `side-demo-delta-0-voxel-0.16.png` và `side-demo-delta-1.73-voxel-0.16.png`: A chỉ có 1 hộp nhỏ ở x≈13m, B có 13 hộp rải khắp scene. Hộp vehicles đầu tiên trong B (x≈8.1m, score=0.93) không xuất hiện trong A — cho thấy delta ảnh hưởng lớn đến khả năng nhận diện của model, không chỉ dịch vị trí.
- **Diễn giải phép z thuận/ngược:** Khi đưa vào model: `z_model = z_source - z_ground - delta`. Khi xuất hộp: `z_source = z_model + z_ground + delta`. Delta=1.73 giả định cao độ sensor KITTI. Nếu quên phép ngược (cộng lại z_ground+delta), hộp sẽ chìm xuống ~1.805m — đúng như case-batch-z cho thấy.
- **Một quyết định lỗi batch và hành động:** Ở case-batch-z, cả 13 hộp đều lệch −1.805m, class/x/y/yaw không đổi → đây là lỗi pipeline (quên phép chuyển ngược), không phải lỗi đối tượng → quyết định: dừng, không sửa tay, báo LC kiểm pipeline. Ở case-one-box-z, chỉ 1 hộp lệch → kiểm từng đối tượng bằng nhiều view.
- **Điều chưa chắc:** Chưa rõ tại sao A (delta=0) chỉ nhận 1 hộp trong khi cùng PCD và checkpoint — có thể do input không đúng phân bố z mà model mong đợi, nhưng không đủ cơ sở khẳng định chính xác cơ chế bên trong model. Cũng chưa chắc pillar 0.32 luôn cho kết quả kém hay chỉ kém vì checkpoint không train cho pillar này.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
