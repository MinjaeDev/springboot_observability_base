# Spring Boot 운영 안정성 공통 기반

> 멀티 서버 환경에서 API 요청/응답 추적,
> 예외 처리 일원화, 실시간 장애 알림을 위한 공통 인프라

---

## 배경

도메인별 7개 독립 Spring Boot 서버를 운영하는 환경에서
서버마다 API 응답 형식과 예외 처리 방식이 제각각이었다.
운영 중 장애가 발생해도 어떤 요청이 언제 들어왔는지
추적할 수 없었고, 예외 발생을 즉각 인지하기도 어려웠다.
이를 해결하기 위해 공통 기반을 직접 설계하고 전 서버에 적용했다.

---

## 문제 상황

**1. 요청/응답 추적 불가**
HTTP 요청 본문(InputStream)은 한 번만 읽을 수 있는 구조적 한계가 있었다.
인터셉터에서 Request Body를 읽으면 실제 컨트롤러에서
Body가 비어있는 문제가 발생했다.

**2. 예외 처리 분산**
예외 로직이 서비스 레이어 곳곳에 흩어져 있어 코드 중복이 심했다.
우리은행 결제 콜백 API처럼 외부 금융사 스펙에 맞는
별도 응답 포맷이 필요한 케이스도 일관되게 처리할 수 없었다.

**3. 장애 인지 지연**
운영 중 예외가 발생해도 로그를 직접 확인하기 전까지
문제를 인지할 방법이 없었다.

---

## 설계 결정

### 왜 Filter + Interceptor 조합인가

Filter 단계에서 Request/Response를 `ContentCachingWrapper`로 감싸
여러 번 읽을 수 있도록 처리하고,
Interceptor에서 캐싱된 본문을 안전하게 읽는 구조를 선택했다.
이 방식은 Spring의 처리 파이프라인을 그대로 유지하면서
로깅 책임만 분리할 수 있다.

### 왜 GlobalExceptionHandler에서 응답 포맷을 분기하는가

우리은행 InfoPlus API는 외부 금융사 스펙상
일반 API와 다른 응답 포맷(`DepositCallbackApiResponse`)을 요구했다.
예외 유형별 처리 로직은 공통화하되,
URI 기반으로 응답 포맷만 분기하는 방식으로
코드 중복 없이 두 스펙을 동시에 만족했다.

---

## 핵심 구조
```
HTTP 요청
    ↓
[BodyWrappingFilter]
ContentCachingRequestWrapper / ContentCachingResponseWrapper로 래핑
    ↓
[LoggerInterceptor - preHandle]
[API START] 메서드 + URI 로깅
    ↓
컨트롤러 처리
    ↓
[LoggerInterceptor - afterCompletion]
Request Body + Response Body 로깅
    ↓
예외 발생 시 → [GlobalExceptionHandler]
    ├── URI가 결제 콜백이면 → DepositCallbackApiResponse 반환
    └── 일반 API면 → ApiResult 반환
    └── 모든 예외 → Slack 자동 알림
```

---

## 핵심 코드

### BodyWrappingFilter
```java
@Component
@RequiredArgsConstructor
public class BodyWrappingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // InputStream을 여러 번 읽을 수 있도록 래핑
        ContentCachingRequestWrapper wrappedRequest =
                new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper wrappedResponse =
                new ContentCachingResponseWrapper(response);

        try {
            filterChain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            // 응답 본문을 클라이언트에 전달
            wrappedResponse.copyBodyToResponse();
        }
    }
}
```

### LoggerInterceptor
```java
@Slf4j
@Component
public class LoggerInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response, Object handler) {
        log.info("[API START] {} {}", request.getMethod(), request.getRequestURI());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler, Exception ex) {
        // 캐싱된 Request Body 읽기
        if (request instanceof ContentCachingRequestWrapper cachingRequest) {
            String requestBody = new String(
                    cachingRequest.getContentAsByteArray(), StandardCharsets.UTF_8);
            log.info("[REQUEST BODY] {}", requestBody);
        }

        // 캐싱된 Response Body 읽기
        if (response instanceof ContentCachingResponseWrapper cachingResponse) {
            String responseBody = new String(
                    cachingResponse.getContentAsByteArray(), StandardCharsets.UTF_8);
            log.info("[API END] {} {}, resp: {}",
                    request.getMethod(), request.getRequestURI(), responseBody);
        }
    }
}
```

### GlobalExceptionHandler (핵심 부분)
```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 결제 콜백 API는 외부 금융사 스펙에 맞는 별도 응답 포맷으로 분기
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<?> handleBusinessException(
            BusinessException e, HttpServletRequest request) {

        if (isSpecialApiRequest(request.getRequestURI())) {
            // 우리은행 InfoPlus 콜백 스펙 응답
            return ResponseEntity.status(e.getStatus())
                    .body(DepositCallbackApiResponse.fail(
                            e.getMessage(),
                            request.getHeader("requestid")));
        }
        // 일반 API 공통 응답
        return ResponseEntity.status(e.getStatus())
                .body(ApiResult.fail(e.getMessage()));
    }

    // 모든 예외 발생 시 Slack 자동 알림
    private void sendSlackErrorNotification(
            Exception e, HttpServletRequest request) {
        String errorMessage = String.format(
                "🚨 [서버 에러 발생]\n" +
                "- 발생 시간: %s\n" +
                "- 예외 타입: %s\n" +
                "- 요청 URL: %s\n" +
                "- 클라이언트 IP: %s",
                Instant.now(),
                e.getClass().getName(),
                request.getRequestURI(),
                request.getRemoteAddr());
        slackService.message(errorMessage);
    }

    private boolean isSpecialApiRequest(String uri) {
        return uri.equals("/pw/ua/infoplus/deposit") ||
               uri.equals("/pw/ua/infoplus/paymentCallback");
    }
}
```

### ApiResult — 공통 응답 래퍼
```java
@Getter
@AllArgsConstructor
@NoArgsConstructor
public class ApiResult<T> {
    private boolean result;
    private String message;
    private T body;
    private long timestamp;

    public static <T> ApiResult<T> success(T body) {
        return new ApiResult<>(true, "success", body, System.currentTimeMillis());
    }

    public static <T> ApiResult<T> fail(String message) {
        return new ApiResult<>(false, message, null, System.currentTimeMillis());
    }
}
```

---

## 처리하는 예외 유형

| 예외 | HTTP 상태 |
|---|---|
| `BusinessException` | 도메인별 정의 |
| `MethodArgumentNotValidException` | 400 |
| `HttpMessageNotReadableException` | 400 |
| `MissingRequestHeaderException` | 400 |
| `SizeLimitExceededException` | 413 |
| `NoResourceFoundException` | 404 |
| `HttpRequestMethodNotSupportedException` | 405 |
| `Exception` (기타 모든 예외) | 500 |

---

## 결과

| 지표 | 개선 전 | 개선 후 |
|---|---|---|
| 장애 원인 파악 시간 | 서버 직접 접속 후 로그 탐색 | 로그에서 즉시 요청/응답 확인 |
| 장애 인지 방식 | 사용자 제보 후 인지 | Slack 알림으로 즉각 인지 |
| 장애 원인 파악 시간 단축 | - | **약 30% 단축** |
| 적용 서버 수 | - | **6개 서버 공통 적용** |

---

## 기술 스택

- **Language**: Java 21
- **Framework**: Spring Boot 3.x
- **예외 처리**: `@RestControllerAdvice`
- **로깅**: `HandlerInterceptor` + `ContentCachingWrapper`
- **알림**: Slack Webhook
- **API 문서**: SpringDoc OpenAPI (Swagger)
