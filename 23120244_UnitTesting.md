# Báo cáo lab Nhóm 09

**Họ tên:** Nguyễn Nhật Duy · **MSSV:** 23120244 · **JDK đã dùng** (kết quả `java -version`): openjdk version "17.0.19" 2026-04-21 (Temurin-17.0.19+10)  
**Link repo:** [nhatduy27/UnitTestExercise](https://github.com/nhatduy27/UnitTestExercise)

## Bài 1: Giá trị biên

| Ngày trễ | Hạng | Số tiền bạn điền | Cách tính |
|---:|---|---:|---|
| 24 | NORMAL | 48000 | 24 * 2.000 = 48.000 |
| 25 | NORMAL | 50000 | 25 * 2.000 = 50.000 (đạt mức trần tối đa) |
| 26 | NORMAL | 50000 | 26 * 2.000 = 52.000 -> lấy mức trần 50.000 |
| 2 | VIP | 0 | Trong 2 ngày ân hạn (miễn phí) |
| 3 | VIP | 1000 | Ngày thứ 3 (trừ 2 ngày ân hạn còn 1 ngày tính phí): 1 * 1.000 = 1.000 |
| 51 | VIP | 49000 | (51 - 2) * 1.000 = 49.000 |
| 52 | VIP | 50000 | (52 - 2) * 1.000 = 50.000 (đạt mức trần tối đa) |
| 53 | VIP | 50000 | (53 - 2) * 1.000 = 51.000 -> lấy mức trần 50.000 |

**Vì sao chọn các ngày này mà không chọn ngày 10?**
Vì kiểm thử giá trị biên (Boundary Value Analysis) tập trung vào các điểm chuyển giao logic và giới hạn tính toán - nơi dễ phát sinh lỗi lập trình nhất (lỗi off-by-one như `<`, `<=`, `>`, `>=`). Cụ thể:
- Với NORMAL: ngày 25 là biên đạt mức phạt tối đa 50.000đ; chọn 24 (ngay dưới biên), 25 (tại biên), 26 (ngay trên biên).
- Với VIP: ngày 2 và 3 là biên kết thúc thời gian ân hạn (ngày 2 miễn phí, ngày 3 bắt đầu tính tiền); ngày 51, 52, 53 là biên chạm trần phạt 50.000đ ((52 - 2) * 1.000 = 50.000).
Ngày 10 chỉ là một giá trị thông thường nằm giữa phân vùng tương đương, không kiểm tra được các điều kiện ranh giới nhạy cảm này.

## Bài 2a: Coverage

| | Branch coverage của `LoanService` |
|---|---|
| Trước khi viết test | 66% (4/6 branches) |
| Sau khi viết 2 test | 100% (6/6 branches) |

**Test `daDatHanMuc` của bạn dùng `countActiveByMemberId` trả về giá trị nào? Vì sao chọn giá trị đó?**
Trả về giá trị 3L. Vì theo đặc tả "mỗi thành viên được mượn tối đa 3 cuốn: đang mượn 3 cuốn thì không được mượn thêm", do đó 3 chính là giá trị biên nhỏ nhất tại đó hệ thống phải từ chối yêu cầu mượn và ném mã lỗi LIMIT_REACHED.

## Bài 2b: Mutation testing

**Lần chạy PIT đầu tiên:** có bao nhiêu mutant sống sót (SURVIVED)? Nằm ở dòng nào, là loại mutant gì?
Có 1 mutant sống sót (SURVIVED) nằm ở dòng 28 trong LoanService.java. Loại mutant: "changed conditional boundary" (PIT đã thay đổi toán tử điều kiện biên nhưng test không phát hiện ra nếu mock giá trị 4L).

**Lỗi cài sẵn nằm ở đâu? Bạn phát hiện ra nó bằng cách nào?**
Lỗi cài sẵn nằm ở dòng 28 file `LoanService.java`: `if (loans.countActiveByMemberId(memberId) > 3)`. Phát hiện bằng cách:
1. PIT báo mutant "changed conditional boundary" sống sót tại dòng này khi test với 4L.
2. Đối chiếu với đặc tả: "đang mượn 3 cuốn thì không được mượn thêm", tức là đang có 3 cuốn là đã không được mượn tiếp. Khi viết test mock giá trị biên là 3 cuốn (`thenReturn(3L)`), test bị fail vì code `> 3` vẫn cho phép mượn tiếp (3 > 3 là false).

**Bạn đã sửa test và code như thế nào?**

1. **Sửa code trong `LoanService.java` (dòng 28):**
   - *Trước khi sửa (sai nghiệp vụ biên):*
     ```java
     if (loans.countActiveByMemberId(memberId) > 3) {
         throw new LoanException(LIMIT_REACHED);
     }
     ```
   - *Sau khi sửa (đúng đặc tả: đang mượn 3 cuốn thì không được mượn thêm):*
     ```java
     if (loans.countActiveByMemberId(memberId) >= 3) {
         throw new LoanException(LIMIT_REACHED);
     }
     ```

2. **Sửa và hoàn thiện test trong `LoanServiceTest.java`:**
   - *Đoạn test `daDatHanMuc` (dùng giá trị biên `3L` thay vì `4L`):*
     ```java
     // BÀI 2a: thành viên đang mượn đủ số cuốn tối đa thì bị từ chối với mã LIMIT_REACHED.
     @Test void daDatHanMuc() {
         availableBook();
         member(BigDecimal.ZERO);
         when(loans.countActiveByMemberId(1)).thenReturn(3L);
         assertCode(LIMIT_REACHED);
         verify(books, never()).save(any());
     }
     ```
   - *Đoạn test `conPhiChuaTra`:*
     ```java
     // BÀI 2b: thành viên còn nợ phí thì bị từ chối với mã UNPAID_FINE.
     @Test void conPhiChuaTra() {
         availableBook();
         member(BigDecimal.valueOf(10_000));
         when(loans.countActiveByMemberId(1)).thenReturn(0L);
         assertCode(UNPAID_FINE);
         verify(books, never()).save(any());
     }
     ```

**Kết quả PIT sau khi sửa** (Killed bao nhiêu / tổng bao nhiêu):
Killed 9 / tổng 9 (100% mutation coverage).

## Câu hỏi tổng kết
**Coverage 100% có nghĩa là code không còn lỗi không? Hãy giải thích dựa trên những gì bạn thấy trong lab.**
Không. Coverage 100% (cả line coverage và branch coverage) chỉ phản ánh rằng mọi dòng lệnh và mọi nhánh rẽ đã được chạy qua ít nhất một lần bởi bộ test. Nó không đảm bảo rằng kết quả hay điều kiện biên được kiểm tra đúng logic nghiệp vụ. Minh chứng rõ ràng trong bài lab: ban đầu khi test `daDatHanMuc` dùng giá trị 4L, branch coverage vẫn đạt 100%, nhưng code thực tế bị lỗi nghiệp vụ ở biên (`> 3` thay vì `>= 3`). Lỗi này chỉ bị phát hiện khi áp dụng Mutation Testing hoặc khi áp dụng kỹ thuật kiểm thử giá trị biên chính xác.
