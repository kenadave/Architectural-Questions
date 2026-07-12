@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/auth/**", "/h2-console/**", "/actuator/health").permitAll()
                        .anyRequest().denyAll()
                )
                // Allow H2 console frames (dev only — disable in production)
                .headers(headers -> headers.frameOptions(fo -> fo.sameOrigin()))
                .build();
    }

    /**
     * BCrypt with default strength (10 rounds).
     * Complies with secure coding rule: use bcrypt/scrypt/Argon2 with unique salt.
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

--------------------------------------------------------------------------------------------------------------------
@Configuration — tells Spring this class declares beans. Without it, @Bean methods would not be picked up.

@EnableWebSecurity — activates Spring Security's web support. Without it, 
Spring Boot's auto-configuration would apply default rules (which includes HTTP Basic auth on everything). 
This annotation hands you full control and disables the defaults.

--------------------------------------------------------------------------------------------------------------------
@Bean works on methods in any Spring-managed class — @Configuration, @Component, @Service, @RestController. All of them work. But the behaviour is different.

With @Configuration (full mode):

Spring subclasses your configuration class using CGLIB (a proxy). Every @Bean method call goes through the proxy, which checks if the bean already exists in the context before creating a new one.


@Configuration
public class AppConfig {

    @Bean
    public A a() {
        return new A(b()); // Spring intercepts b() — returns the SAME instance
    }

    @Bean
    public B b() {
        return new B();    // only created once
    }
}
b() is called inside a(), but Spring intercepts that call and returns the existing singleton bean — b() code runs only once.

--------------------------------------------------------------------------------------------------------------------
With @Component (lite mode):

No CGLIB proxy. @Bean methods are plain Java methods. Calling one from another creates a new instance — it bypasses the container.


@Component
public class AppConfig {

    @Bean
    public A a() {
        return new A(b()); // plain Java call — creates a NEW B, not the bean
    }

    @Bean
    public B b() {
        return new B();    // creates another NEW B
    }
}

Two different B objects — not the same singleton. This causes subtle bugs.

Rule: Use @Configuration whenever your @Bean methods call each other. Use @Component only when they are fully independent.


--------------------------------------------------------------------------------------------------------------------

@Service and @Repository are specialisations of @Component with additional meaning — @Service signals business logic, 
@Repository signals data access and enables exception translation. They behave identically to @Component at the Spring level but carry semantic intent.

--------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------

Authentication FLow:

Phase 1 — Registration (one time)
Client
POST /auth-service/auth/register
Body: { email, password, role: "CUSTOMER" }


JwtAuthenticationFilter >  Request passes straight through to auth-service without any token check for Public paths: Login, Register, and Health check
--------------------------------------------------------------------------------------------------------------------

Phase 2 — Login (subsequent sessions)

Client
POST /auth-service/auth/login
Body: { email, password }

--------------------------------------------------------------------------------------------------------------------

Phase 3 — Authenticated request (every subsequent call)
Client now has the token. It wants to create a booking:

Client
POST /booking-service/bookings
Headers: Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.xxxxx.zzzzz
Body: { flightId, firstName, ... }

// 1. Path check — not public, proceed with validation
isPublicPath("/booking-service/bookings") → false

// 2. Read Authorization header
String authHeader = "Bearer eyJhbGciOiJIUzI1NiJ9.xxxxx.zzzzz"

// 3. Strip "Bearer " prefix
String token = "eyJhbGciOiJIUzI1NiJ9.xxxxx.zzzzz"

// 4. Validate — JwtTokenValidator.isValid()
Jwts.parser()
    .verifyWith(secretKey)       // same secret as auth-service
    .build()
    .parseSignedClaims(token)    // verifies signature + expiry
    // throws JwtException if tampered or expired → 401

--------------------------------------------------------------------------------------------------------------------

The full picture in one diagram


                    ┌──────────────────────────────────────────────────┐
                    │                  API GATEWAY                      │
POST /auth/register │                                                   │
────────────────────▶  CorrelationIdFilter      (mint trace ID)        │
or /auth/login      │  JwtAuthenticationFilter  (public path → skip)  │
                    │  RoleAuthorizationFilter  (public path → skip)  ├──▶ auth-service
                    │                                                   │    BCrypt hash
                    │                                                   │    generateToken()
                    │                                                   │    ← JWT
                    └──────────────────────────────────────────────────┘
                              ← JWT returned to client

                    ┌──────────────────────────────────────────────────┐
                    │                  API GATEWAY                      │
POST /booking       │                                                   │
Authorization:      │  CorrelationIdFilter      (mint trace ID)        │
Bearer <token> ─────▶  JwtAuthenticationFilter  (validate signature)  │
                    │    valid   → extract userId + role               │
                    │    invalid → 401 stop                            │
                    │  RoleAuthorizationFilter  (check role)           ├──▶ booking-service
                    │    allowed → forward with X-User-Id/Role         │    reads X-User-Id
                    │    denied  → 403 stop                            │    enforceOwnership()
                    └──────────────────────────────────────────────────┘
a    



RegisterRequest / LoginRequest
        │
        ▼
AuthController → AuthService
                     │ BCrypt.encode / BCrypt.matches
                     │ UserRepository (DB)
                     ▼
              JwtTokenProvider.generateToken()
                     │ HMAC-SHA256(payload, JWT_SECRET)
                     ▼
              AuthResponse (token) → client

─────────────────────────────────────────────

Client sends: Authorization: Bearer <token>
        │
        ▼
JwtAuthenticationFilter
        │ JwtTokenValidator.parse()
        │ HMAC-SHA256 verify + expiry check
        │ extract userId + role
        ▼
RoleAuthorizationFilter
        │ check role against path rules
        ▼
BookingController (@RequestHeader X-User-Id, X-User-Role)
        │
        ▼
BookingService.enforceOwnership()

