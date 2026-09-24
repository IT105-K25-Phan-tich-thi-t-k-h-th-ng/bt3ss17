# BÁO CÁO PHÂN TÍCH VÀ THIẾT KẾ ĐẶC TẢ PHÂN HỆ TÍNH CƯỚC VẬN ĐƠN RIKKEILOGISTICS

> 👤 **Học viên:** Đỗ Hoàng Sơn | **Mã SV:** PTIT-HCM-066
> 🏫 **Môn học:** IT105-K25-Phan-tich-thi-t-k-h-th-ng

---

## 📊 Sơ đồ thiết kế hệ thống (Activity Diagram)

> 💡 *Sơ đồ dưới đây được render tự động trực tiếp trên GitHub bằng Mermaid. Bạn cũng có thể tải file **`bt3.drawio`** trong repository này để mở và chỉnh sửa trực tiếp trên [Draw.io (diagrams.net)](https://app.diagrams.net).* 

```mermaid
graph TD
  Start([Bắt đầu nhập liệu]) --> InputCheck{Kiểm tra định dạng Input Kích thước & Cân nặng?}
  InputCheck -- Lỗi: Chữ cái, số âm, >150cm --> ErrInput[Báo lỗi: Vi phạm kích thước băng chuyền] --> EndFail([Kết thúc: Từ chối vận đơn])
  InputCheck -- Hợp lệ --> WeightCalc[Tính khối lượng quy đổi: Dài x Rộng x Cao / 5000]
  WeightCalc --> CompareWeight[Khối lượng tính cước = Max(Thực tế, Quy đổi)]
  CompareWeight --> ServiceSelect{Chọn loại dịch vụ vận chuyển?}
  ServiceSelect -- Economy --> CalcEco[Cước cơ bản: 15.000đ/1kg đầu + 5.000đ/0.5kg tiếp]
  ServiceSelect -- Express --> CalcExp[Cước cơ bản: 25.000đ/1kg đầu + 8.000đ/0.5kg tiếp]
  ServiceSelect -- Fast Track --> CalcFast[Cước cơ bản: 45.000đ/1kg đầu + 15.000đ/0.5kg tiếp]
  CalcEco --> PostalCheck{Kiểm tra mã bưu chính vùng sâu/vùng cao/hải đảo?}
  CalcExp --> PostalCheck
  CalcFast --> PostalCheck
  PostalCheck -- Thuộc vùng xa (+20%) --> AddRemote[Cộng phụ phí 20% tổng cước]
  PostalCheck -- Vùng thường --> SkipRemote[Giữ nguyên cước]
  AddRemote --> InsurCheck{Kiểm tra giá trị khai giá > 5.000.000đ?}
  SkipRemote --> InsurCheck
  InsurCheck -- Khai giá lớn --> AddInsur[Cộng phí bảo hiểm 0.5% giá trị khai giá]
  InsurCheck -- Không vượt mức --> SkipInsur[Không cộng phí bảo hiểm]
  AddInsur --> Fork[=== Thanh Fork: Xử lý định tuyến & Lưu cước ===]
  SkipInsur --> Fork
  Fork --> RouteHub[Gợi ý kho trung chuyển gần nhất dựa trên địa chỉ nhận]
  Fork --> SaveDB[Lưu thông tin bưu kiện và cước phí vào Database]
  RouteHub --> Join[=== Thanh Join: Hoàn tất tính cước ===]
  SaveDB --> Join
  Join --> EndSuccess([Kết thúc: Trả về kết quả cước và kho trung chuyển])
```

---

## Phần 1: Lập bảng phân tích Input/Output và ma trận điều kiện tính cước

Trong phần này, tôi tiến hành phân tích chi tiết các luồng dữ liệu vào (Input) và dữ liệu ra (Output) của phân hệ tính cước vận đơn, đồng thời xây dựng ma trận điều kiện xử lý các bẫy dữ liệu và quy tắc nghiệp vụ theo đúng yêu cầu của bài toán RikkeiLogistics.

| Thành phần | Tên biến / Trường dữ liệu | Kiểu dữ liệu | Ràng buộc & Quy tắc kiểm tra (Validation) |
| --- | --- | --- | --- |
| Input | Dài, Rộng, Cao | Float / Number | Phải là số dương (>0). Bất kỳ chiều nào vượt 150cm sẽ kích hoạt cơ chế từ chối do vượt chuẩn băng chuyền. |
| Input | Khối lượng thực tế | Float / Number | Phải là số dương (>0). Dùng để so sánh với khối lượng quy đổi thể tích. |
| Input | Loại dịch vụ | Enum | Giá trị bắt buộc thuộc một trong ba loại: 'Economy', 'Express', 'Fast Track'. |
| Input | Mã bưu chính / Địa chỉ | String | Kiểm tra tính hợp lệ của mã bưu chính. Nếu thuộc vùng sâu/vùng cao/hải đảo sẽ áp dụng phụ phí 20%. |
| Input | Giá trị khai giá | Float / Number | Số tiền khai báo bưu kiện. Nếu > 5.000.000đ bắt buộc cộng thêm 0.5% phí bảo hiểm. |
| Output | Khối lượng tính cước | Float / Number | Kết quả Max(Khối lượng thực tế, Khối lượng quy đổi theo công thức (Dài x Rộng x Cao) / 5000). |
| Output | Cước vận chuyển tổng cộng | Float / Number | Tổng tiền cước gồm cước cơ bản theo dịch vụ + phụ phí vùng sâu (nếu có) + phí bảo hiểm (nếu có). |
| Output | Kho trung chuyển gợi ý | String | Tên mã kho trung chuyển tối ưu dựa trên địa chỉ người nhận. |

## Phần 2: Activity Diagram chuẩn UML cho phân hệ tính cước

Sơ đồ Activity Diagram được thiết kế toàn diện bằng mã Mermaid ở trên, thể hiện chính xác logic xử lý từ bước nhận diện bẫy dữ liệu kích thước vượt chuẩn, tính toán khối lượng quy đổi thể tích, phân rã theo gói dịch vụ, kiểm tra phụ phí vùng miền, tính bảo hiểm khai giá, cho đến cơ chế xử lý song song (Fork/Join) giữa việc gợi ý kho trung chuyển và lưu trữ dữ liệu vào cơ sở dữ liệu.

- Sử dụng khối rẽ nhánh (Decision Node) để kiểm tra các bẫy dữ liệu đầu vào như kích thước âm, chữ cái hoặc vượt giới hạn 150cm của băng chuyền.
- Áp dụng chính xác công thức quy đổi thể tích (Dài x Rộng x Cao) / 5000 và hàm Max để xác định trọng lượng tính cước.
- Tách rõ các mức cước cơ bản cho ba loại dịch vụ Economy, Express và Fast Track.
- Xử lý song song (Fork/Join) cho tác vụ định tuyến kho và lưu database giúp hệ thống tối ưu hóa hiệu năng.

## Phần 3: Đặc tả REQ-CALC-01 theo chuẩn IEEE 830

Dưới đây là đặc tả chi tiết yêu cầu chức năng cho phân hệ tính cước vận chuyển tự động, được biên soạn gọn gàng, đảm bảo tính rõ ràng (Unambiguous) và có thể kiểm thử được (Verifiable).

| Thành phần đặc tả | Nội dung chi tiết cho REQ-CALC-01 |
| --- | --- |
| Mô tả chức năng | Hệ thống tự động nhận các thông số kích thước (Dài, Rộng, Cao), khối lượng thực tế, loại dịch vụ, mã bưu chính và giá trị khai giá để tính toán chính xác cước phí vận chuyển, phụ phí vùng miền, phí bảo hiểm và gợi ý kho trung chuyển phù hợp. |
| Input / Output | Input tham chiếu từ Bảng phân tích Phần 1 (Kích thước, Khối lượng thực tế, Loại dịch vụ, Mã bưu chính, Giá trị khai giá). Output trả về Khối lượng tính cước, Tổng cước vận chuyển và Kho trung chuyển gợi ý. |
| Điều kiện tiên quyết (Pre-conditions) | Người dùng hoặc hệ thống đối tác phải truyền đầy đủ các trường thông tin bắt buộc, định dạng dữ liệu hợp lệ (số dương, mã bưu chính tồn tại trong hệ thống danh mục). Bưu kiện phải vượt qua kiểm tra giới hạn băng chuyền (<= 150cm mỗi chiều). |

---

## 📁 Danh sách tệp tin nộp bài trong Repository
- 📝 `bt3.docx`: Báo cáo tài liệu phân tích nghiệp vụ hoàn chỉnh.
- 🎨 `bt3.drawio`: File thiết kế sơ đồ chuẩn theo quy định đề bài (mở trực tiếp bằng [Draw.io](https://app.diagrams.net) hoặc Lucidchart).
