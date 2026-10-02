# common-logging

Thư viện logging dùng chung cho các service trong `xxx-market`: annotation
`@Logging` (AOP, ghi log request/response theo format chuẩn), correlation-id
filter (MDC: traceId, clientIp, sourceIp, forwardId, referer, device,
channel, agent, endpoint), và mask field nhạy cảm (password, token,...).

Tự động cấu hình (Spring Boot auto-configuration) — chỉ cần thêm dependency
là dùng được, không cần khai báo bean nào thêm.

## Cài đặt

```xml
<dependency>
    <groupId>com.xxx</groupId>
    <artifactId>xxx-common-logging</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

Yêu cầu: service phải là Spring Boot web app (servlet-based, có
`spring-boot-starter-web`) trên Spring Boot 3.4.x / Java 21.

Build & install lib vào local repo (chưa publish lên Nexus):

```bash
cd xxx-common-logging
./mvnw clean install
```

## Dùng `@Logging`

```java
import com.xxx.common.logging.annotation.Logging;

@RestController
public class MyController {

    @Logging("get-orders")
    @GetMapping("/orders")
    public List<Order> getOrders(...) { ... }
}
```

Mỗi lần method chạy sẽ ghi 1 dòng log (logger tên cấu hình qua
`xxx.logging.logger-name`, mặc định `xxx_LOG`) là **một
object JSON** gồm các field: `action, endpoint (method+path+query), traceId,
status (httpStatus thật), durationMs, clientIp, sourceIp, forwardId, referer,
device, channel, agent, args, result, error`. `args`/`result` được nhúng dạng
JSON lồng (không phải chuỗi đã escape) nên dòng log luôn là JSON hợp lệ, kể cả
khi giá trị có khoảng trắng, dấu `=`, hay xuống dòng (ví dụ thông báo lỗi
nhiều dòng).

- `result` (response payload) **chỉ log khi logger ở mức DEBUG** — mặc định
  INFO chỉ có metadata gọn (không có `result`), tránh log quá nặng.
- Field trong `xxx.logging.mask-fields` (mặc định:
  `password,token,secret,authorization`) tự động bị mask thành `***` trong
  cả `args` và `result`. Có thể thêm field riêng cho từng method:
  `@Logging(maskFields = {"cardNumber"})`.

## Muốn field `status` phản ánh đúng HTTP status khi lỗi

Nếu exception của service bạn implement `HttpStatusAware`, `LoggingAspect`
sẽ log đúng status đó thay vì mặc định `500`:

```java
public class MyBusinessException extends RuntimeException implements HttpStatusAware {
    @Override
    public int httpStatus() { return 404; }
}
```

## Cấu hình (application.yaml)

`logger-name` nên đặt riêng cho từng ứng dụng (ví dụ theo tên service) vì
thư viện này dùng chung cho nhiều app — để mỗi app tự định tuyến dòng log
này sang appender/logger riêng của mình mà không đụng app khác.

```yaml
logging:
  level:
    ORDER_SERVICE_ACCESS_LOG: INFO   # đổi thành DEBUG để log đầy đủ response payload

xxx:
  logging:
    logger-name: ORDER_SERVICE_ACCESS_LOG   # mặc định: xxx_LOG
    mask-fields: password,token,secret,authorization,cardNumber
```

## Correlation ID

`CorrelationIdFilter` tự động chạy trước mọi filter khác (kể cả Spring
Security nếu có), gắn header response `X-Request-Id` và đẩy toàn bộ context
vào MDC — nên các JSON log appender khác (nếu dùng `logstash-logback-encoder`)
cũng tự động có các field này mà không cần cấu hình gì thêm.
