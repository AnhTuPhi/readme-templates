# XXX verify gate (POC)

Service trung gian để các hệ thống legacy của XXX (Core CK, Web, CRM…) xác thực CCCD gắn chip
qua dịch vụ eID của Trung tâm RAR, thay vì mỗi hệ thống tự tích hợp AgentGW.
Legacy chỉ gửi `idCard` + `dsCert` + `address`, nhận về `verified: true/false`.
Phần ký RSA, `request_time`, `license_id`, `device_type` và chuẩn hoá mã lỗi nằm trong service này.

- Đặc tả POC: [`docs/verify-gate-poc-spec.md`](docs/verify-gate-poc-spec.md)
- Sơ đồ kiến trúc: [`docs/XXX-verify-gate-architecture.html`](docs/XXX-verify-gate-architecture.html)
- Postman: [`docs/XXX-verify-gate.postman_collection.json`](docs/XXX-verify-gate.postman_collection.json)

## Kiến trúc

```
Hệ thống legacy (Core CK, Web, CRM…)
   │  POST /api/v1/verify/verify   {idCard, dsCert, address, purpose, location, source}
   ▼
┌── XXX-verify-gate (1 jar, stateless, không DB) ───────────────────────────────┐
│  CorrelationIdFilter (traceId)  ─  XXX-common-logging                          │
│  controller   VerifyController      @Valid + @Logging (access log, che PII)     │
│       ▼                                                                         │
│  service      verifyVerificationService   request → VerifyCommand → response │
│       ▼       verifyVerifier (interface)  ◄── điểm cắm VNeID pha sau          │
│  provider/eid EidVerifier                                                       │
│                 ├─ kiểm tra đầu vào trước khi ký (Base64, ký tự '|', xuống dòng) │
│                 ├─ signature/CskSigner         ký SHA256withRSA, Base64 1 dòng   │
│                 ├─ client/AgentGwClient        HTTP, giữ body raw               │
│                 ├─ signature/EidResponseVerifier  verify chữ ký RAR (bật/tắt)   │
│                 └─ mapper/EidErrorMapper       VRF_* / HTTP / exitcode → ErrorCode │
│  model/exception  GlobalExceptionHandler      VerifyException → HTTP + ErrorResponse │
└─────────────────────────────────────────────────────────────────────────────────┘
   │ HTTPS 443 (sandbox) / HTTP :8443 (AgentGW)   header: request_id, request_time,
   ▼                                              license_id, device_type, dia_chi, signature
AgentGW (RAR, VLAN riêng) ── VPN site-to-site ──► EID Platform (RAR) ──► C06
```

### Luồng xử lý một request

1. `CorrelationIdFilter` gắn `traceId` (lấy từ header `X-Request-Id` hoặc tự sinh), trả lại ở response.
2. `VerifyController` validate `VerifyRequest`: idCard đúng 12 số, dsCert là Base64 một dòng,
   address ≤ 255 ký tự… Sai thì trả 400 `INPUT_INVALID`, **không gọi AgentGW**.
3. `verifyVerificationService` chuyển request thành `VerifyCommand` (không phụ thuộc provider)
   và gọi `verifyVerifier`.
4. `EidVerifier`:
   - kiểm tra thêm trước khi ký: dsCert decode được, không có `|` hay ký tự điều khiển;
   - sinh `request_id` (UUID v4) và `request_time` (`yyyyMMddHHmmss`, GMT+7);
   - ký chuỗi `request_id|request_time|license_id|device_type|dia_chi|id_card|dscert`;
   - gửi AgentGW và ghi 2 dòng log `EID >>` / `EID <<` cùng `requestId`.
5. Phản hồi HTTP 200 → `verified` lấy từ `responds.result`. Phản hồi lỗi → `EidErrorMapper`
   chuẩn hoá thành `ErrorCode` và ném `VerifyException`.
6. `GlobalExceptionHandler` trả `ErrorResponse` với HTTP status tương ứng. Access log của
   `@Logging` ghi đúng status nhờ `VerifyException implements HttpStatusAware`.

## Cấu trúc mã nguồn

Luồng phụ thuộc một chiều: `controller → service → provider`. `model`, `config` và `util` dùng chung.

```
vn.com.XXX.verifygate
├── verifygateApplication
├── config                     Cấu hình Spring
│   ├── EidProperties            @ConfigurationProperties("eid")
│   ├── EidConfig                bean khoá CSK / public key RAR, RestClient tới AgentGW
│   └── ProviderDirEnvironment   tự tìm thư mục providers/ trước khi nạp cấu hình
├── controller                 Nhận/trả HTTP, không chứa nghiệp vụ
│   ├── VerifyController         POST /v1/verify/verify
│   └── HealthController         GET  /v1/health
├── service                    Điều phối nghiệp vụ
│   ├── verifyVerificationService   request API → VerifyCommand → response API (pha sau: chọn provider)
│   └── verifyVerifier              interface cho mọi provider (eID, VNeID…)
├── model
│   ├── VerifyCommand, VerifyResult   dữ liệu trao đổi giữa service và provider
│   ├── request/VerifyRequest         DTO vào của API
│   ├── response/                     DTO ra: VerifyResponse, HealthResponse, ErrorResponse
│   ├── eid/                          định dạng dữ liệu AgentGW: EidRequestBody, EidResponds, EidErrorBody
│   └── exception/                    ErrorCode, VerifyException, GlobalExceptionHandler
├── provider
│   └── eid                    Tích hợp eID (Trung tâm RAR)
│       ├── EidVerifier          triển khai verifyVerifier
│       ├── client/              AgentGwClient
│       ├── signature/           CskSigner, EidResponseVerifier, RawJsonExtractor
│       └── mapper/              EidErrorMapper
└── util
    └── PemKeyLoader           đọc khoá PEM (PKCS#8 / SubjectPublicKeyInfo) bằng JDK thuần
```

Thư mục ngoài mã nguồn:

```
providers/eid/eid.yml          cấu hình eID (không chứa secret)
providers/eid/keys/            khoá .pem, private key không commit
docs/                          đặc tả, sơ đồ kiến trúc, Postman collection
src/main/resources/            application.yml, logback-spring.xml, META-INF/spring.factories
```

Test đặt cùng package với class được test (`src/test/java/...`).

**Thêm provider mới (vd. VNeID):**
1. Viết `provider/vneid/VneidVerifier implements verifyVerifier`.
2. Tạo `providers/vneid/vneid.yml` + `keys/`, thêm dòng import vào `spring.config.import` trong `application.yml`.
3. Cho `verifyVerificationService` chọn verifier theo mã provider.

Không phải sửa controller hay DTO API.

## Yêu cầu

- Java 21, Maven (dùng `./mvnw`)
- Thư viện `com.XXX:XXX-common-logging:1.0.0-SNAPSHOT` trong Maven repo
  (chưa có trên Nexus: `cd XXX-common-logging && ./mvnw clean install`)
- Máy chạy phải **đồng bộ NTP**: AgentGW từ chối `request_time` lệch quá 10 phút hoặc ở tương lai
- Có firewall/VPN tới AgentGW: Sandbox trả `nginx 403` nếu IP chưa được whitelist
- `file.encoding` = UTF-8 (mặc định từ JDK 18; service tự kiểm tra khi khởi động).
  Header `dia_chi` có dấu phải được gửi đúng byte UTF-8 đã ký.

## Cấu hình

Cấu hình và khoá của từng provider nằm **ngoài jar**, trong thư mục `providers/`
(xem [`providers/README.md`](providers/README.md)). `application.yml` nạp `${PROVIDER_DIR}/eid/eid.yml`.

Nếu không khai báo `PROVIDER_DIR`, `ProviderDirEnvironment` tự tìm `providers/` theo thứ tự:
working directory → thư mục chứa jar/classes và tối đa 3 cấp cha. Vì vậy chạy từ IDE,
`java -jar target/verify-gate.jar` từ bất kỳ đâu, hay deploy `/opt/verify-gate/{verify-gate.jar, providers/}`
đều không cần cấu hình thêm. Đường dẫn tìm được in ra log lúc khởi động (`Provider dir: ...`).

| Biến môi trường | Mặc định | Ghi chú |
|---|---|---|
| `PROVIDER_DIR` | tự tìm | Thư mục cấu hình + khoá các provider |
| `EID_BASE_URL` | `https://sandbox-eid.congxacthucdientuquocgia.gov.vn` | Hoặc `http://<AgentGW>:8443`, chờ RAR xác nhận |
| `EID_LICENSE_ID` | `LIC_TEST_001` | |
| `EID_DEVICE_TYPE` | `XXX_DEVICE_TYPE_TEST` | **Giá trị tạm**, phải thay bằng giá trị RAR cấp |
| `EID_CSK_KEY` | `<PROVIDER_DIR>/eid/keys/cks_prive_key_sandbox.pem` | Private key CSK, PKCS#8 (`BEGIN PRIVATE KEY`) |
| `EID_RAR_PUB` | `<PROVIDER_DIR>/eid/keys/public_sandbox.pem` | Public key RAR để verify response, chỉ nạp khi bật cờ dưới |
| `EID_VERIFY_RESPONSE_SIGNATURE` | `false` | RAR chưa công bố quy tắc tuần tự hoá `responds` |
| `SERVER_PORT` | `8080` | Context path `/api` |
| `LOG_PATH` | `./logs` | Thư mục file log |

Private key đặt quyền `chmod 600`. **Không commit**: `.gitignore` chặn `*.pem`, chỉ cho phép
`providers/*/keys/public_*.pem`.

> `public_sandbox.pem` của RAR **không** phải cặp của `cks_prive_key_sandbox.pem` (đã đối chiếu modulus).
> Đó là khoá RAR dùng ký response. Muốn tự verify chữ ký request offline, sinh public key từ chính
> private key: `openssl pkey -in cks_prive_key_sandbox.pem -pubout`.

## Chạy local

```bash
./mvnw test                 # 96 unit test, không cần mạng
./mvnw package -DskipTests  # → target/verify-gate.jar

# chép cks_prive_key_sandbox.pem vào providers/eid/keys/ rồi chạy
EID_DEVICE_TYPE=<do RAR cấp> java -jar target/verify-gate.jar
```

Chạy trong IntelliJ: run config `verifygateApplication`, thêm `EID_DEVICE_TYPE` vào Environment variables.

```bash
curl http://localhost:8080/api/v1/health

curl -X POST http://localhost:8080/api/v1/verify/verify \
  -H 'Content-Type: application/json; charset=utf-8' \
  -H 'X-Client-Id: core-ck' \
  --data-binary @request.json      # file UTF-8; trên Windows đừng gõ tiếng Việt trực tiếp trong -d
```

`request.json`:

```json
{"idCard":"001099012345","dsCert":"MIIEr...","address":"Hà Nội","purpose":"OPEN_ACCOUNT","location":"HO-HN","source":"core-ck"}
```

## API

### `POST /api/v1/verify/verify`

| Trường | Bắt buộc | Mô tả | Gửi sang AgentGW |
|---|---|---|---|
| `idCard` | x | Số CCCD, đúng 12 chữ số | body `id_card` |
| `dsCert` | x | Chứng thư số trên chip CCCD, Base64 một dòng | body `dscert` |
| `address` | x | Tỉnh/thành thường trú trên căn cước, ≤ 255 ký tự | header `dia_chi` |
| `purpose` | | Mục đích xác thực, ≤ 255 ký tự | body `authentication_purpose` |
| `location` | | Địa điểm khởi tạo, ≤ 255 ký tự | body `authentication_location` |
| `source` | | Nguồn/kênh khởi tạo, ≤ 255 ký tự | body `authentication_source` |

Header `X-Client-Id` (tuỳ chọn) chỉ dùng để ghi log, POC chưa xác thực client.

Thành công (HTTP 200):

```json
{"verified":true,"message":"valid DSCert","providerRefId":"test_service-...","requestId":"<uuid>","signatureVerified":null}
```

- `verified` lấy từ `responds.result` (exitcode 0 vẫn có thể `result=false` với DSCert giả).
- `requestId` là `request_id` đã gửi sang AgentGW, dùng để đối soát log với RAR.
- `signatureVerified` = `null` khi tắt verify chữ ký response.
- Header response `X-Request-Id` là `traceId` của gate, tra được trên mọi dòng log.

Lỗi: `{"code":"...","message":"...","providerCode":"VRF_...","requestId":"..."}`.
`providerCode` là mã gốc của eID; `requestId` chỉ có khi request đã được gửi sang AgentGW.

| code | HTTP | Retry | Nguồn từ eID |
|---|---|---|---|
| `INPUT_INVALID` | 400 | Không | Validation tại gate, `VRF_INVALID_*`, `VRF_MISSING_HEADER`, exitcode 1–5 |
| `SIGNATURE_INVALID` | 502 | Không, cảnh báo ngay | `BAD_REQUEST` "Xác thực chữ ký đối tác thất bại", 401 |
| `CLIENT_NOT_AUTHORIZED` | 502 | Không | 403 `FORBIDDEN`, exitcode 7–11, nginx 403 (chưa whitelist IP) |
| `DUPLICATED_REQUEST` | 409 | Không | 409 `CONFLICT` |
| `RATE_LIMITED` | 429 | Có | `VRF_RATE_LIMITED`, 429 |
| `PROVIDER_UNAVAILABLE` | 503 | Có | `VRF_PLATFORM_*`, 502/504, không kết nối được |
| `PROVIDER_INTERNAL` | 502 | **Không** | `VRF_CRYPTO_FAIL`, `VRF_INTERNAL`, 500, `BAD_REQUEST` lệch `request_time` |
| `SIGNING_FAILED` / `INTERNAL_ERROR` | 500 | Không | Lỗi nội bộ gate |
| `RESPONSE_SIGNATURE_INVALID` | 502 | Không | Verify chữ ký response thất bại |

Chi tiết ánh xạ: `EidErrorMapper` và `EidErrorMapperTest`.

### `GET /api/v1/health`

```json
{"status":"UP","provider":"EID","baseUrl":"https://sandbox-eid.congxacthucdientuquocgia.gov.vn"}
```

## Test bằng Postman

Import [`docs/XXX-verify-gate.postman_collection.json`](docs/XXX-verify-gate.postman_collection.json):
1 request `POST /v1/verify/verify` để test thông luồng gate → AgentGW (RAR).
Đặt biến `baseUrl` (mặc định `http://localhost:8080/api`), `idCard`, `dsCert` (bộ dữ liệu kiểm thử RAR cấp), `address`.

| Test | Ý nghĩa |
|---|---|
| Thông luồng tới AgentGW | Request đã tới AgentGW và nhận phản hồi eID (HTTP 200 hoặc lỗi có mã gốc eID) |
| Có requestId để đối soát với RAR | gate đã ký và gửi request |
| HTTP 200 và verified = true | Dữ liệu kiểm thử được RAR xác thực |

Khi lỗi, Console in gợi ý nguyên nhân (bị chặn IP/VPN, chữ ký, lệch giờ, license, device_type...).

```bash
npx newman run docs/XXX-verify-gate.postman_collection.json \
  --env-var baseUrl=http://localhost:8080/api \
  --env-var idCard=<id_card RAR> --env-var dsCert=<dscert RAR>
```

## Log

Dùng chuẩn chung `XXX-common-logging` + `src/main/resources/logback-spring.xml`:

| File (trong `LOG_PATH`) | Nội dung |
|---|---|
| `XXX-verify-gate-access.log` | Access log chuẩn XXX từ `@Logging("verify-verify")`: 1 dòng JSON/request (traceId, status, durationMs, clientIp, request đã che…) |
| `XXX-verify-gate.log` | Log ứng dụng, gồm đúng 2 dòng `EID >>` / `EID <<` mỗi lần gọi AgentGW (cùng `requestId`) |
| `XXX-verify-gate-error.log` | Chỉ WARN/ERROR |

- Tên file lấy theo `spring.application.name`; logger access log là `XXX_verify_gate_LOG`.
- `traceId` có trên mọi dòng log.
- Che dữ liệu theo `XXX.logging.mask-fields` (`idCard, dsCert, id_card, dscert, signature`…) thành `******`,
  áp dụng cho cả access log và body gửi AgentGW. Không log khoá.
- Profile `prod` không ghi log ứng dụng ra console. Đặt `logging.level.XXX_verify_gate_LOG: DEBUG`
  để log cả response.
- File archive nén theo ngày/kích thước tại `LOG_PATH/archive/` (`LOG_MAX_FILE_SIZE`, `LOG_MAX_HISTORY`,
  `LOG_TOTAL_SIZE_CAP`).

## Câu hỏi còn mở với RAR

1. Quy tắc tuần tự hoá `responds` để verify chữ ký response.
2. Endpoint chính thức: `http://<AgentGW>:8443` hay HTTPS 443 domain sandbox.
3. Giá trị `device_type` cấp cho XXX; bộ dữ liệu kiểm thử `id_card` + `dscert` + `dia_chi`.
4. Quy cách `dia_chi`: có dấu hay không dấu, có tiền tố "Thành phố/Tỉnh" không. Nếu có dấu, AgentGW đọc
   byte UTF-8 hay yêu cầu dạng khác (percent-encode)? Service hiện gửi đúng byte UTF-8 đã ký.
5. Lỗi chữ ký: tài liệu ghi 401, thực tế trả 400 `BAD_REQUEST`.
