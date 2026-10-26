# NHIỆM VỤ: Nâng cấp hệ thống nghiên cứu thuật toán dự đoán Vietlott và XSMB

## 1. Mục tiêu

Đọc và phân tích toàn bộ mã nguồn hiện tại của dự án trước khi sửa. Xây dựng hệ thống tự động thử nghiệm, huấn luyện, đánh giá và so sánh nhiều thuật toán thống kê/học máy cho cả ba bài toán:

1. Vietlott Power 6/55.
2. Vietlott Mega 6/45.
3. XSMB (xổ số miền Bắc).

Mục tiêu là kiểm tra xem các thuật toán có cải thiện kết quả dự đoán ngoài mẫu so với baseline hay không, không mặc định rằng thuật toán phức tạp hơn sẽ tốt hơn.

Không viết lại toàn bộ dự án nếu không cần thiết. Tái sử dụng kiến trúc, dữ liệu, cơ chế walk-forward, SQLite, sensor, giao diện web và các dịch vụ Home Assistant đang có.

## 2. Khảo sát trước khi sửa

Đọc các file hiện có, tối thiểu gồm những thành phần liên quan đến:

- Tải, chuẩn hóa và lưu dữ liệu lịch sử.
- Tạo features.
- Thuật toán và quá trình huấn luyện.
- Walk-forward backtest.
- Cập nhật dự đoán trực tuyến.
- SQLite và cơ chế chống xử lý trùng.
- Home Assistant sensor, service và giao diện web.
- Cấu hình add-on, phiên bản, dependency và test.

Xác định chính xác cấu trúc thư mục và luồng dữ liệu của Vietlott và XSMB. Không giả định hai hệ thống dùng chung schema, cách tính xác suất hoặc quy tắc kết quả.

Trước khi sửa, lập danh sách file cần thay đổi và giải thích ngắn gọn. Sau đó triển khai trực tiếp, không chỉ trả về hướng dẫn hoặc pseudocode.

## 3. Thuật toán cần bổ sung

Tạo giao diện chung để đăng ký và chạy thuật toán, nhưng giữ riêng target, feature, cách tính xác suất và luật chấm điểm cho từng trò chơi.

### Nhóm A — Baseline và xác suất

- Uniform probability: xác suất đồng đều làm baseline bắt buộc.
- Empirical frequency.
- Beta-Binomial hoặc Bayesian smoothing.
- Rolling frequency với nhiều cửa sổ lịch sử.
- EWMA với nhiều hệ số suy giảm.

### Nhóm B — Học máy

- Regularized Logistic Regression.
- SGDClassifier với loss và cách hiệu chỉnh xác suất phù hợp.
- HistGradientBoosting hoặc một mô hình boosting phù hợp với schema dữ liệu.
- Ensemble kết hợp các mô hình nếu có bằng chứng rằng các mô hình bổ sung thông tin cho nhau.
- Hedge hoặc online weighting chỉ khi cập nhật trọng số đúng thứ tự thời gian và không dùng kết quả tương lai.

Không bắt buộc phải dùng mọi thuật toán nếu không phù hợp với dữ liệu hoặc target. Nếu dependency mới là cần thiết, giải thích lý do, cân nhắc khả năng tương thích với môi trường add-on và cập nhật đúng file cấu hình.

## 4. Quy tắc riêng cho từng trò chơi

### Vietlott Power 6/55

- Dự đoán sáu số chính được rút từ 1–55.
- Xử lý số Power riêng, không trộn vào target sáu số chính.
- Chỉ đánh giá số Power nếu đã xác định đúng dữ liệu lịch sử, luật trò chơi và mục tiêu riêng.
- Bảo đảm phân phối dự đoán phù hợp với ràng buộc của trò chơi.

### Vietlott Mega 6/45

- Dự đoán sáu số chính từ 1–45.
- Bảo đảm mô hình không dự đoán số ngoài miền hợp lệ.
- Đánh giá theo cùng các kỳ và cùng quy tắc cho tất cả mô hình.

### XSMB

- Đọc schema kết quả hiện tại để xác định chính xác các giải, hai số cuối và loto.
- Giữ riêng các bài toán như dự đoán hai số cuối giải đặc biệt, tần suất loto hoặc các target khác đang được hỗ trợ.
- Không gộp các target khác nhau vào cùng một mô hình nếu cách sinh dữ liệu và cách chấm điểm khác nhau.
- Không tự ý đổi định nghĩa target hoặc cấu trúc sensor hiện có.
- Không dùng dữ liệu giải hoặc kỳ quay chưa xảy ra để tạo feature.

## 5. Walk-forward bắt buộc, chống data leakage

Mọi mô hình phải được đánh giá theo thời gian thực giả lập:

1. Sắp xếp dữ liệu theo ngày/kỳ và loại bỏ bản ghi trùng theo quy tắc đã xác minh.
2. Dùng warm-up phù hợp với lượng dữ liệu.
3. Ở kỳ t, chỉ dùng dữ liệu có sẵn trước kỳ t để tạo features, huấn luyện, hiệu chuẩn và lựa chọn tham số.
4. Tạo và lưu dự đoán cho kỳ t trước khi đọc hoặc ghi nhận kết quả kỳ t.
5. Khi kết quả kỳ t đã có, mới tính điểm và cập nhật trạng thái online.
6. Tiếp tục đến kỳ cuối, giữ nguyên thứ tự thời gian.
7. Dùng cùng tập kỳ đánh giá khi so sánh các mô hình trên cùng một target.

Không được huấn luyện trên toàn bộ lịch sử rồi dùng mô hình đó để chấm lại các kỳ quá khứ. Không được tính trọng số ensemble bằng toàn bộ kết quả backtest rồi áp dụng ngược cho những kỳ đã chấm.

Tách rõ hai chế độ:

- **Backtest:** tái hiện quá trình dự đoán quá khứ theo đúng thứ tự thời gian.
- **Latest training:** huấn luyện bằng toàn bộ dữ liệu hiện có để tạo dự đoán cho kỳ tiếp theo.

Việc lựa chọn mô hình hoặc hyperparameter phải thực hiện bên trong dữ liệu quá khứ, không được sử dụng tập kỳ đánh giá cuối cùng để chọn mô hình. Nếu có đủ dữ liệu, sử dụng giai đoạn xác nhận theo thời gian và giữ lại một giai đoạn đánh giá cuối chưa dùng để lựa chọn.

## 6. Feature engineering

Nghiên cứu và thử các feature chỉ sử dụng dữ liệu quá khứ:

- Tần suất xuất hiện theo nhiều cửa sổ.
- Số kỳ kể từ lần xuất hiện gần nhất.
- EWMA và rolling frequency.
- Phân bố chẵn/lẻ, tổng các số và các thống kê mô tả phù hợp với target.
- Tần suất cặp số hoặc nhóm số nếu có cơ sở thống kê và đủ dữ liệu.
- Các feature riêng cho XSMB dựa trên lịch sử kết quả của đúng target.

Không tạo quá nhiều feature tùy tiện. Ghi lại feature set, tham số và phiên bản code của mỗi thí nghiệm để có thể tái lập kết quả.

## 7. Đánh giá mô hình

Mỗi target phải có baseline đồng đều hoặc baseline đúng theo phân phối lý thuyết của trò chơi.

Tính các metric phù hợp:

- Log-loss.
- Brier score.
- Số lượng số trùng top-k.
- Số kỳ đạt các mức trùng khác nhau nếu phù hợp.
- Độ hiệu chuẩn xác suất.
- Kết quả theo các giai đoạn thời gian.
- Khoảng tin cậy cho chênh lệch metric giữa mô hình và baseline.

Chuẩn hóa metric đúng với target. Ghi rõ Log-loss/Brier là trung bình theo số, theo kỳ hay theo xác suất của cả tổ hợp; không so sánh trực tiếp các metric dùng định nghĩa khác nhau.

Dùng paired bootstrap theo khối thời gian hoặc phương pháp kiểm định phù hợp để đánh giá độ bất định. Điều chỉnh việc thử nhiều mô hình/hyperparameter nhằm hạn chế chọn nhầm mô hình tốt do ngẫu nhiên.

Không tự động đánh dấu mô hình tốt hơn chỉ vì top-k cao hơn. Chỉ xem xét cải thiện khi kết quả ngoài mẫu ổn định, có ý nghĩa thống kê và không gây suy giảm đáng kể các metric xác suất khác.

Nếu không mô hình nào vượt baseline đáng tin cậy, giữ baseline làm chuẩn và ghi nhận rõ kết quả đó.

## 8. Hệ thống thí nghiệm tự động

Xây dựng module quản lý thí nghiệm để:

- Đăng ký danh sách thuật toán và cấu hình.
- Tạo các thí nghiệm với tham số được giới hạn.
- Chạy backtest cho từng trò chơi và target.
- Ghi log lỗi, thời gian chạy, phiên bản code, feature set, tham số và metric.
- So sánh mô hình trên cùng kỳ đánh giá.
- Lưu bảng xếp hạng và lý do chấp nhận hoặc loại bỏ.
- Hỗ trợ giới hạn số thí nghiệm, thời gian chạy và bộ nhớ để không ảnh hưởng Home Assistant.

Nếu tích hợp LLM, chỉ cho LLM đọc báo cáo, đề xuất thuật toán, feature và cấu hình thí nghiệm trong phạm vi cho phép. Không cho phép LLM tự sửa database, bỏ qua kiểm định hoặc thay mô hình đang vận hành mà không qua quy trình phê duyệt.

Không bắt buộc phải dùng API AI trong mỗi lần chạy. Toàn bộ pipeline backtest phải chạy được bằng Python khi không có kết nối mạng sau khi dữ liệu và dependency đã sẵn sàng.

## 9. SQLite và vận hành an toàn

- Giữ nguyên database và dữ liệu hiện tại.
- Không xóa, reset hoặc ghi đè dữ liệu đang vận hành.
- Xem xét migration có phiên bản nếu cần thêm bảng hoặc cột.
- Dùng transaction cho thao tác ghi liên quan.
- Đảm bảo restart add-on không làm mất trạng thái hoặc tạo dự đoán trùng.
- Nếu lịch sử cũ thay đổi, phát hiện và báo lỗi rõ ràng; không âm thầm tiếp tục bằng trạng thái không nhất quán.
- Không chạy các thí nghiệm nặng trong đường xử lý sự kiện cần phản hồi nhanh.
- Không để lỗi của một mô hình làm dừng toàn bộ các mô hình hoặc dịch vụ khác.
- Cung cấp khả năng rollback về phiên bản mô hình trước đó.

## 10. Giao diện và báo cáo

Cập nhật giao diện hiện có để hiển thị theo từng trò chơi và target:

- Danh sách mô hình.
- Trạng thái chạy, lỗi và thời điểm chạy gần nhất.
- Số kỳ train/warm-up và số kỳ backtest.
- Log-loss, Brier, top-k và baseline tương ứng.
- Chênh lệch metric so với baseline, khoảng tin cậy và kết quả theo thời gian.
- Tham số, feature set và phiên bản mô hình.
- Lý do mô hình được chấp nhận hoặc bị loại.
- Thời điểm cập nhật dữ liệu và dự đoán gần nhất.

Không làm mất hoặc thay đổi không cần thiết các sensor, service, URL và giao diện hiện tại. Nếu cần thêm entity, tuân theo quy ước đặt tên đang dùng.

## 11. Kiểm thử bắt buộc

Thêm test cho cả Vietlott và XSMB, tối thiểu gồm:

- Xác suất đầu ra hợp lệ, không NaN/Inf, không âm và đúng miền số.
- Xử lý dữ liệu thiếu, trùng, sai định dạng và kết quả bị sửa.
- Chứng minh không có data leakage bằng test có dữ liệu tương lai giả lập.
- Dự đoán được lưu trước khi kết quả được ghi nhận.
- Không chấm cùng một kỳ hai lần.
- Restart có thể khôi phục trạng thái.
- Backtest có thể tái lập khi dùng cùng dữ liệu, cấu hình và seed.
- Một mô hình lỗi không làm hỏng pipeline chung.
- Mỗi mô hình được so sánh trên cùng kỳ với baseline.
- Migration SQLite và cấu hình add-on hoạt động với dữ liệu cũ.

Chạy toàn bộ test hiện có và test mới; chạy lint, compile, kiểm tra cấu hình và `git diff --check` theo công cụ dự án đang sử dụng.

## 12. Điều kiện hoàn thành

Chỉ kết thúc khi:

1. Cả ba bài toán Power 6/55, Mega 6/45 và các target XSMB hiện có đều có pipeline thí nghiệm phù hợp.
2. Các thuật toán mới thực sự chạy được trên dữ liệu lịch sử; không tạo kết quả giả để minh họa.
3. Báo cáo có baseline, metric, số kỳ đánh giá và định nghĩa metric rõ ràng.
4. Walk-forward không có data leakage.
5. Không phá vỡ tính tương thích với Home Assistant và dữ liệu hiện có.
6. Tất cả test hiện có và test mới đều đạt, hoặc báo cáo chính xác test nào thất bại và nguyên nhân.
7. Không tự ý commit, push, triển khai hoặc ghi đè dữ liệu đang chạy.

Cuối cùng, hãy cung cấp:
- Tóm tắt các file đã thay đổi.
- Danh sách thuật toán thực sự được triển khai.
- Bảng kết quả backtest thực tế cho từng trò chơi/target.
- So sánh với baseline và kết luận có hay không có cải thiện đáng tin cậy.
- Các lệnh đã chạy và kết quả kiểm thử.
- Các hạn chế còn tồn tại.

Không tuyên bố mô hình dự đoán được tốt hơn ngẫu nhiên nếu dữ liệu kiểm định không chứng minh điều đó. Ưu tiên tính đúng đắn, khả năng tái lập và an toàn vận hành hơn số lượng thuật toán.