# Bài 4: Thực hành Viết Unit Test

## 1. Phác thảo các kịch bản kiểm thử biên cần thiết

### Cho TransactionParser

Cần bao phủ các trường hợp sau:

- JSON rỗng hoặc null
  - Parser phải trả về danh sách rỗng.
- JSON sai định dạng
  - Parser không ném exception, mà trả về danh sách rỗng hoặc xử lý an toàn.
- Amount âm hoặc bằng 0
  - Dòng giao dịch bị bỏ qua.
- Amount là chuỗi chữ không hợp lệ
  - Dòng giao dịch bị bỏ qua.
- Status không hợp lệ
  - Dòng bị bỏ qua.
- JSON đúng định dạng
  - Phải map đúng các trường vào `TransactionDTO`.

### Cho LedgerBalanceCalculator

Cần bao phủ các trường hợp sau:

- Danh sách tài khoản null
  - Trả về 0.
- Danh sách rỗng
  - Trả về 0.
- Tài khoản null trong list
  - Bị bỏ qua.
- Tài khoản có số dư âm
  - Không được tính.
- activeOnly = true
  - Chỉ tính tài khoản trạng thái `ACTIVE`.
- activeOnly = false
  - Tính tất cả tài khoản có số dư dương.

---

## 2. Prompt kiểm thử logic do tôi thiết kế

```text
Bạn là Senior QA Engineer chuyên Java và testing.

Hãy sinh bộ unit test cho 2 class sau:
1. TransactionParser
2. LedgerBalanceCalculator

Yêu cầu công nghệ:
- Sử dụng JUnit 5.
- Sử dụng Mockito cho các dependency cần mock.
- Dùng annotation @Mock và @InjectMocks khi có thể.
- Viết code sạch, dễ đọc, có chú thích bằng tiếng Việt.

Kịch bản kiểm thử cần bao phủ:
- TransactionParser:
  - chuỗi JSON rỗng;
  - JSON bị lỗi định dạng;
  - JSON chứa số tiền âm;
  - JSON đúng định dạng.
- LedgerBalanceCalculator:
  - danh sách Account null;
  - danh sách rỗng;
  - danh sách chứa tài khoản số dư âm;
  - danh sách chứa tài khoản null;
  - kiểm tra lọc theo trạng thái hoạt động (activeOnly).

Yêu cầu đầu ra:
- File test cho TransactionParser.
- File test cho LedgerBalanceCalculator.
- Mã nguồn JUnit 5 hoàn chỉnh, có chú thích tiếng Việt và dễ chạy trong môi trường Maven/Gradle.
```

---

## 3. Minh chứng chạy thực tế

### Log phản hồi AI (mô phỏng)

```text
Đây là bộ test JUnit 5 hoàn chỉnh cho cả hai class.

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class TransactionParserTest {

    @Mock
    private ObjectMapper objectMapper;

    @InjectMocks
    private TransactionParser transactionParser;

    @Test
    void shouldReturnEmptyListWhenRawJsonIsBlank() {
        List<TransactionDTO> result = transactionParser.parse("   ");
        assertTrue(result.isEmpty());
    }

    @Test
    void shouldReturnEmptyListWhenJsonIsMalformed() throws Exception {
        when(objectMapper.readTree("{bad json}"))
                .thenThrow(new com.fasterxml.jackson.core.JsonParseException(null, "Malformed JSON"));

        List<TransactionDTO> result = transactionParser.parse("{bad json}");
        assertTrue(result.isEmpty());
    }

    @Test
    void shouldSkipTransactionsWithNegativeAmount() throws Exception {
        String rawJson = "[{\"id\":\"T1\",\"amount\":\"-10\",\"status\":\"SUCCESS\",\"transactionDate\":\"2024-01-01T10:00:00\"}]";
        ObjectMapper realMapper = new ObjectMapper().registerModule(new JavaTimeModule());
        TransactionParser parser = new TransactionParser(realMapper);

        List<TransactionDTO> result = parser.parse(rawJson);
        assertTrue(result.isEmpty());
    }

    @Test
    void shouldParseValidTransaction() throws Exception {
        String rawJson = "[{\"id\":\"T2\",\"amount\":\"150.50\",\"status\":\"FAILED\",\"transactionDate\":\"2024-02-02T12:30:00\"}]";
        ObjectMapper realMapper = new ObjectMapper().registerModule(new JavaTimeModule());
        TransactionParser parser = new TransactionParser(realMapper);

        List<TransactionDTO> result = parser.parse(rawJson);

        assertEquals(1, result.size());
        assertEquals("T2", result.get(0).id());
        assertEquals(new BigDecimal("150.50"), result.get(0).amount());
        assertEquals("FAILED", result.get(0).status());
        assertEquals(LocalDateTime.of(2024, 2, 2, 12, 30), result.get(0).localDateTime());
    }
}
```

```java
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class LedgerBalanceCalculatorTest {

    @Test
    void shouldReturnZeroWhenAccountListIsNull() {
        LedgerBalanceCalculator calculator = new LedgerBalanceCalculator();
        assertEquals(0.0, calculator.calculateTotalBalance(null, "HCM", true));
    }

    @Test
    void shouldReturnZeroWhenAccountListIsEmpty() {
        LedgerBalanceCalculator calculator = new LedgerBalanceCalculator();
        assertEquals(0.0, calculator.calculateTotalBalance(List.of(), "HCM", true));
    }

    @Test
    void shouldIgnoreAccountsWithNegativeBalance() {
        LedgerBalanceCalculator calculator = new LedgerBalanceCalculator();
        List<Account> accounts = List.of(
                new Account("A1", "HCM", "ACTIVE", -100.0),
                new Account("A2", "HCM", "ACTIVE", 200.0)
        );

        double result = calculator.calculateTotalBalance(accounts, "HCM", true);
        assertEquals(200.0, result);
    }

    @Test
    void shouldIgnoreNullAccounts() {
        LedgerBalanceCalculator calculator = new LedgerBalanceCalculator();
        List<Account> accounts = List.of(null, new Account("A3", "HCM", "ACTIVE", 100.0));

        double result = calculator.calculateTotalBalance(accounts, "HCM", true);
        assertEquals(100.0, result);
    }

    @Test
    void shouldFilterOnlyActiveAccountsWhenActiveOnlyIsTrue() {
        LedgerBalanceCalculator calculator = new LedgerBalanceCalculator();
        List<Account> accounts = List.of(
                new Account("A1", "HCM", "ACTIVE", 100.0),
                new Account("A2", "HCM", "INACTIVE", 200.0)
        );

        double result = calculator.calculateTotalBalance(accounts, "HCM", true);
        assertEquals(100.0, result);
    }
}
```
```
