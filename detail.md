# Giải thích theo từng bước

## Phần A: Phân biệt class của thư viện và class tự viết

### Class/interface của thư viện

| Thành phần | Thư viện | Package | Vai trò |
|---|---|---|---|
| `Authentication` (interface) | Spring Security | `org.springframework.security.core` | Đại diện cho "một yêu cầu xác thực" hoặc "kết quả đã xác thực" |
| `UsernamePasswordAuthenticationToken` | Spring Security | `org.springframework.security.authentication` | Implementation của `Authentication`, chứa username/password |
| `AuthenticationManager` (interface) | Spring Security | `org.springframework.security.authentication` | Cổng vào của quá trình xác thực, chỉ có 1 method `authenticate()` |
| `ProviderManager` | Spring Security | `org.springframework.security.authentication` | Implementation mặc định của `AuthenticationManager`, giữ danh sách `AuthenticationProvider` và duyệt qua từng cái |
| `DaoAuthenticationProvider` | Spring Security | `org.springframework.security.authentication.dao` | Provider xác thực bằng username/password, lấy user qua `UserDetailsService` |
| `UserDetailsService` (interface) | Spring Security | `org.springframework.security.core.userdetails` | Chỉ có `loadUserByUsername()`, nơi bạn nối vào DB |
| `UserDetails` (interface) | Spring Security | `org.springframework.security.core.userdetails` | Đối tượng trả về từ `UserDetailsService` (username, password đã mã hóa, authorities) |
| `PasswordEncoder` (interface) | Spring Security Crypto | `org.springframework.security.crypto.password` | Mã hóa và so khớp mật khẩu |
| `BCryptPasswordEncoder` | Spring Security Crypto | `org.springframework.security.crypto.bcrypt` | Implementation dùng thuật toán BCrypt |
| `SecurityContext` (interface) | Spring Security | `org.springframework.security.core.context` | Hộp chứa `Authentication` của người dùng hiện tại |
| `SecurityContextHolder` | Spring Security | `org.springframework.security.core.context` | Lưu `SecurityContext` theo thread (ThreadLocal) |
| `SecurityContextRepository` (interface) | Spring Security | `org.springframework.security.web.context` | Quy định cách lưu và đọc `SecurityContext` giữa các request |
| `HttpSessionSecurityContextRepository` | Spring Security | `org.springframework.security.web.context` | Implementation lưu `SecurityContext` vào `HttpSession` |
| `SecurityContextHolderFilter` | Spring Security | `org.springframework.security.web.context` | Filter đọc `SecurityContext` từ repository ở đầu mỗi request |
| `BadCredentialsException` | Spring Security | `org.springframework.security.authentication` | Exception khi sai username/password |
| `JdbcIndexedSessionRepository` | Spring Session JDBC | `org.springframework.session.jdbc` | Lưu session xuống DB, thay cho session trong bộ nhớ của Tomcat |
| `SessionRepositoryFilter` | Spring Session | `org.springframework.session.web.http` | Filter bọc `HttpServletRequest`, để `request.getSession()` thực chất đi vào DB |

### Class tự viết

| Thành phần | Vai trò |
|---|---|
| `AuthController` | Nhận request `POST /auth/login` |
| `AuthService` | Điều phối: tạo token, gọi `AuthenticationManager`, lưu `SecurityContext` |
| `LoginRequest` (record) | DTO chứa `username`, `password` |
| `CustomUserDetailsService` | Implement `UserDetailsService`, đọc user từ DB qua `UserRepository` |
| `UserRepository` | Repository (thường extends `JpaRepository`, thuộc Spring Data JPA) |
| `User` | Entity ánh xạ bảng user |
| `GlobalExceptionHandler` | `@RestControllerAdvice` bắt exception và trả về JSON lỗi |

---

## Phần B: Cấu hình ban đầu (khi ứng dụng khởi động)

Bước này chạy một lần, trước khi có request nào.

**Bước B1. Khai báo `PasswordEncoder` bean**
Bạn tạo bean `PasswordEncoder` trả về `new BCryptPasswordEncoder()`. Interface là `PasswordEncoder`, class thực tế là `BCryptPasswordEncoder`.

**Bước B2. Spring tự dựng `AuthenticationManager`**
Khi có bean `UserDetailsService` (chính là `CustomUserDetailsService`) và bean `PasswordEncoder`, Spring Security tự tạo một `DaoAuthenticationProvider`, gắn hai bean trên vào, rồi bọc nó trong một `ProviderManager`. Bạn lấy nó ra bằng `AuthenticationConfiguration.getAuthenticationManager()` và khai báo thành bean để inject vào `AuthService`.

**Bước B3. Khai báo `SecurityContextRepository` bean**
Tạo bean `new HttpSessionSecurityContextRepository()`. Đây là class quyết định "lưu SecurityContext vào HttpSession".

**Bước B4. Bật Spring Session JDBC**
Thêm dependency `spring-session-jdbc`. Spring Boot tự đăng ký `SessionRepositoryFilter` và `JdbcIndexedSessionRepository`, đồng thời tạo bảng `SPRING_SESSION` và `SPRING_SESSION_ATTRIBUTES`. Từ đây mọi `HttpSession` sẽ nằm trong DB.

---

## Phần C: Luồng đăng nhập `POST /auth/login`

**Bước 1. Client gửi request**
Client gửi JSON `{ "username": "...", "password": "..." }`. Trước khi vào controller, request đi qua `SessionRepositoryFilter` (Spring Session). Filter này bọc request/response để mọi lời gọi `getSession()` sau đó đều đi qua `JdbcIndexedSessionRepository`.

**Bước 2. `AuthController` nhận request** *(tự viết)*
Spring parse JSON thành `LoginRequest` (record), rồi controller gọi `authService.login(request, httpReq, httpRes)`. Controller không chứa logic bảo mật.

**Bước 3. `AuthService` tạo token chưa xác thực** *(tự viết, dùng class thư viện)*
Gọi `UsernamePasswordAuthenticationToken.unauthenticated(username, password)`. Token này implement `Authentication`, trong đó principal là username dạng String, credentials là mật khẩu thô và `isAuthenticated() = false`.

**Bước 4. `AuthService` gọi `AuthenticationManager.authenticate(token)`**
Biến được khai báo bằng interface `AuthenticationManager`, nhưng object thật là `ProviderManager`.

**Bước 5. `ProviderManager` chọn provider** *(thư viện)*
`ProviderManager` duyệt danh sách `AuthenticationProvider`, gọi `supports(token.getClass())` trên từng provider. `DaoAuthenticationProvider` hỗ trợ `UsernamePasswordAuthenticationToken` nên nó được chọn, và `ProviderManager` ủy quyền cho nó.

**Bước 6. `DaoAuthenticationProvider` gọi `UserDetailsService`** *(thư viện gọi code của bạn)*
Nó gọi `loadUserByUsername(username)`. Đây là điểm mở rộng chính: đối tượng thật là `CustomUserDetailsService` do bạn viết.

**Bước 7. `CustomUserDetailsService` đọc DB** *(tự viết)*
Gọi `userRepository.findByUsername(username)` để lấy entity `User`. Nếu không thấy, ném `UsernameNotFoundException`. Nếu thấy, chuyển `User` thành `UserDetails` (thường dùng `org.springframework.security.core.userdetails.User` của thư viện, hoặc tự implement) với username, password đã mã hóa BCrypt trong DB, và danh sách quyền (`GrantedAuthority`).

**Bước 8. `DaoAuthenticationProvider` so khớp mật khẩu** *(thư viện)*
Gọi `passwordEncoder.matches(rawPassword, encodedPasswordTrongDB)`. `PasswordEncoder` là interface, thực thi bởi `BCryptPasswordEncoder`. Nó trích salt từ chuỗi hash trong DB, băm lại mật khẩu thô và so sánh.

**Bước 9a. Trường hợp sai thông tin**
`DaoAuthenticationProvider` ném `BadCredentialsException`. Lưu ý: nếu username không tồn tại, `UsernameNotFoundException` cũng được chuyển thành `BadCredentialsException` (mặc định `hideUserNotFoundExceptions = true`) để không lộ thông tin tài khoản nào có thật. Exception đi ngược lên `ProviderManager` → `AuthService` → `AuthController`, rồi `GlobalExceptionHandler` *(tự viết)* bắt lại và trả về `401 Unauthorized`. Luồng kết thúc tại đây.

**Bước 9b. Trường hợp đúng thông tin**
`DaoAuthenticationProvider` tạo một `UsernamePasswordAuthenticationToken` mới, đã xác thực: principal là `UserDetails`, có authorities, `isAuthenticated() = true`. `ProviderManager` xóa credentials (mật khẩu) khỏi token rồi trả về cho `AuthService`.

**Bước 10. `AuthService` tạo `SecurityContext`** *(tự viết, dùng class thư viện)*
```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authentication);
SecurityContextHolder.setContext(context);
```
`SecurityContext` là interface chỉ chứa một `Authentication`. `SecurityContextHolder` giữ nó trong ThreadLocal để các code khác trong cùng request truy cập được.

**Bước 11. `AuthService` lưu context bằng `SecurityContextRepository`**
Gọi `securityContextRepository.saveContext(context, request, response)`. Bước này bắt buộc từ Spring Security 6, vì framework không còn tự lưu khi bạn tự xác thực trong controller.

**Bước 12. `HttpSessionSecurityContextRepository` ghi vào session** *(thư viện)*
Nó gọi `request.getSession(true)` rồi `session.setAttribute("SPRING_SECURITY_CONTEXT", context)`. Vì `request` đã bị `SessionRepositoryFilter` bọc, nên session này không phải của Tomcat mà là `Session` của Spring Session.

**Bước 13. Spring Session ghi xuống DB** *(thư viện)*
Khi response được commit, `JdbcIndexedSessionRepository` thực hiện `INSERT` vào `SPRING_SESSION` (session id, thời gian tạo, hết hạn...) và `SPRING_SESSION_ATTRIBUTES` (attribute `SPRING_SECURITY_CONTEXT` dưới dạng byte đã serialize).

> Vì `SecurityContext` được serialize, các class bên trong nó (`UserDetails`, `GrantedAuthority`...) phải `Serializable`. Nếu bạn tự implement `UserDetails` thì nhớ thêm `implements Serializable`.

**Bước 14. Trả response**
Controller trả `200 OK`, kèm header `Set-Cookie: SESSION=<id>` do Spring Session thêm vào.

---

## Phần D: Các request sau khi đã đăng nhập

**Bước 15. Client gửi request kèm cookie `SESSION`**
`SessionRepositoryFilter` đọc cookie, gọi `JdbcIndexedSessionRepository.findById()` để `SELECT` session từ DB và gắn vào request.

**Bước 16. `SecurityContextHolderFilter` nạp lại context** *(thư viện)*
Filter này gọi `securityContextRepository.loadDeferredContext(request)`. `HttpSessionSecurityContextRepository` đọc attribute `SPRING_SECURITY_CONTEXT` từ session, deserialize ra `SecurityContext`, rồi filter đặt nó vào `SecurityContextHolder`.

**Bước 17. Kiểm tra quyền truy cập**
Các filter phía sau (như `AuthorizationFilter`) lấy `Authentication` từ `SecurityContextHolder` để quyết định cho phép hay chặn (`401`/`403`). Controller của bạn cũng có thể lấy user hiện tại bằng `@AuthenticationPrincipal` hoặc `SecurityContextHolder.getContext().getAuthentication()`.

**Bước 18. Kết thúc request**
`SecurityContextHolderFilter` xóa `SecurityContextHolder` (tránh rò rỉ giữa các thread). Nếu session có thay đổi, Spring Session cập nhật lại DB, ít nhất là `LAST_ACCESS_TIME`.

---

## Tóm tắt nhanh

- Phần **xác thực** (Bước 4 đến 9): `AuthenticationManager` → `ProviderManager` → `DaoAuthenticationProvider` → `UserDetailsService` + `PasswordEncoder`.
- Phần **lưu trạng thái đăng nhập** (Bước 10 đến 13): `SecurityContext` → `SecurityContextRepository` → `HttpSession` → Spring Session JDBC → DB.
- Phần **bạn thực sự phải viết**: `AuthController`, `AuthService`, `CustomUserDetailsService`, `UserRepository`, `User`, `LoginRequest`, `GlobalExceptionHandler` và cấu hình bean. Còn lại là của thư viện.

Nếu muốn, mình có thể viết code Java mẫu cho `SecurityConfig` và `AuthService` khớp từng bước ở trên.
