## 4. Spring Security ##
#### 4.1 Spring Security 활성화 
- 빌드에 의존성 추가하기
    - 스프링 부트 보안 스타터 의존성
    - 보안 테스트 의존성
- 기본적인 보안 기능 환경
    - 모든 HTTP 요청 경로는 인증이 필요
    - 특정 역할, 권한이 없음
    - 로그인 페이지가 없음
    - 스프링 시큐리티의 HTTP 기본 인증을 사용해서 인증됨
    - 단 하나의 사용자(username=user, 암호화된 패스워드 제공)

#### 4.2 Spring Security 구성하기
- SecurityConfig.java
    - 사용자의 HTTP 요청 경로에 대해 접근 제한과 같은 보안 관련 처리 지정
    - WebSecurityConfigurerAdapter(보안 구성 클래스)의 서브 클래스
    - configure(HttpSecurity)
        - HTTP 보안 구성 메서드
    - configure(AuthenticationManagerBuilder)
        - 사용자 인증 정보 구성 메서드
        - 인증을 하기 위해 사용자를 찾는 방법을 지정하는 코드
- 사용자 스토어 구성하기
    - 여러 명의 사용자를 처리할 수 있도록 사용자 정보를 유지 및 관리하는 사용자 스토어(store)
    - Spring Security가 제공하는 사용자 스토어 구성 방법
        1. in-memory 사용자 스토어
            - 변경이 필요 없는 사용자를 정해놓을 때 보안 구성 코드 내부에 정의하여 사용
            - 테스트 목적, 간단한 애플리케이션에 편리
            - 사용자의 추가, 삭제, 변경 시 보안 코드 변경+애플리케이션 재빌드+재배포+재설치 필요
            - 고객 스스로 사용자 등록, 정보 변경에 부적절
            ```java
            @Override
            protected void configure(AuthentictionManagerBuilder auth) throws Exception {
                auth.inMemoryAuthentication()
                .withUser("user1")  //해당 사용자의 구성 시작. username을 인자로 전달
                .password("{noop}password1")  //비밀번호 전달. {noop}으로 비암호화 지정
                .authorities("ROLE_USER");  //.roles("USER")와 같음. 부여 권한 전달
            }
        2. JDBC 기반 사용자 스토어
            - RDBMS로 유지 및 관리 되는 경우 적합
            ```java
            @Autowired
            DataSource dataSource;

            @Override
            protected void configure(AuthenticationManagerBuilder auth) throws Exception {
                auth.jdbcAuthentication()
                .dataSource(dataSource)
                .userByUsernameQuery(
                    "select username, authority " +
                    "from authorities " +
                    "where username=? ")
                .authoritiesByUsernameQuery(
                    "select username, authority " +
                    "from authorities " +
                    "where username=? ")
                .passwordEncoder(new BCryptPasswordEncoder());  //입력받은 비밀번호 암호화
            }
        3. LDAP 기반 사용자 스토어
            - Spring Security의 LDAP 인증에서는 localhost의 33389포트로 LDAP 서버 접속 간주
            - 인증 전략
                - 기본 = 사용자가 직접 LDAP 서버에서 인증받도록
                - 비밀번호 비교 방법(입력된 비밀번호를 LDAP 디렉터리에 전송, 해당 비밀번호를 사용자의 비밀번호 속성 값(userPassword)과 비교하도록 LDAP 서버에 요청→비밀번호 비교는 LDAP 서버에서 수행되어 실제 비밀번호는 노출되지 않음)으로 설정 가능
                - 서버 측에서 비밀번호가 비교될 때는 실제 비밀번호가 서버에 유지된다는 장점 가짐. 단, 사용자로부터 입력 받은 비밀번호는 여전히 해킹의 위험성 가짐 → 암호화로 해결.
            - Spring Security에서 제공하는 내장 LDAP 서버를 의존성 추가하여 사용 가능
            ```java
            @Override
            protected void configure(AuthenticationManagerBuilder auth) throws Exception {
                auth.ldapAuthentication()
                .userSearchBase("ou=people")  //사용자를 찾기 위한 기준점 쿼리 제공.구성단위(ou, organizational unit). 미지정시 LDAP 계층의 루트부터 수행됨
                .userSearchFilter("(uid={0})")
                .groupSearchBase("ou=groups")  //그룹 찾기 위한 기준점 쿼리 지정. 미지정시 LDAP 계층의 루트부터 수행됨
                .groupSearchFilter("member={0}")
                .passwordCompare();  //비밀번호 비교 방법으로 LDAP 인증
                .passwordEncoder(new BCryptPasswordEncoder())
                .passwordAttribute("userPasscode")  //비밀번호 속성 값 지정
                .contextSource().url("ldap://example.com:389/dc=ex,dc=com");  //다른 컴퓨터에서 LDAP 서버 실행중일 떄, 서버 위치 지정
                //.root("dc=ex,dc=com")으로 내장 LDAP 서버 루트 경로 지정 가능
            }
        4. 커스텀 사용자 명세 서비스(Service)
            - 위의 3가지 방법에는 '사용자 이름, 비밀번호, 활성화 여부'의 3가지 정보만 가지고 있어 그 외의 정보가 필요한 경우 커스텀 필요
            - UserDetailService 인터페이스
            ```java
            public interface UserDetailsService {
                UserDetails loadUserByUsername(String username) throws UsernameNotFoundException
                //인자로 전달 받은 사용자 이름으로 찾은 UserDetails 객체가 반환되거나 객체가 없는 경우 예외를 발생시킨다.(null을 반환하지 않는 규칙을 갖는다.)
            }
            ```
            - 커스텀 명세 Service를 Spring Security에 구성해야 한다
            ```java
            public class SecurityConfig extends WebSecurityConfigurerAdapter {
                @Autowired   //자동 주입
                private UserDetailsService userDetailsService;

                @Bean
                public PasswordEncoder encoder() {
                    return new BCryptPasswordEncoder();
                }

                @Override
                protected void configure(AuthenticationManagerBuilder auth) throws Exception {
                    auth.userDeatilsService(userDetailsService)
                    .passwordEncoder(encoder());
                }
            }
            ```
            - 사용자 등록 절차에는 Spring Security가 직접 개입하지 않아 이것을 처리하기 위한 Spring MVC 코드 작성이 필요하다.
#### 4.3 웹 요청 보안(인증) 처리 : configure(HttpSercurity)
- 요청 시 사용자가 합당한 권한을 갖는지 확인하는 것이 가장 많이 일어나는 요소
- HttpSecurity를 사용해 구성할 수 있는 내용
    1. HTTP 요청 처리 허용하기 전에 충족되어야 할 특정 보안 조건 구성
            
        - `규칙을 지정할 떈 순서가 중요하다. 먼저 지정된 경로의 패턴 일치가 먼저 검사된다`
    ```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
        .authorizeRequests()  //url경로와 패턴 및 해당 경로의 보안 요구사항을 구성 가능
        .antMatchers("/design", "/orders")
        .hasRole("ROLE_USER")  //.access("hasRole('ROLE_USER')") 스프링 표현식(SpEL)도 가능
        .antMatchers("/", "/**")
        .permitAll();
    }
    ```

    2. 커스텀 로그인 페이지 구성
        - 커스텀 로그인 페이지 경로를 Spring Security에 지정
        ```java
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.authorizeRequests()
            .antMatchers("/", "/**").permitAll()  //인증 구성 코드
            .and()
            .formLogin()  //커스텀 로그인 폼 구성 코드
            .loginPage("/login")  //경로 지정
            .loginProcessingUrl("/authenticate")  //로그인 요청 처리 경로 지정. 미지정 시 /login이  default.
            .usernameParameter("user")  //사용자 이름 필드 지정. 미지정 시 default값 username
            .passwordParameter("pwd")  //비밀번호 필드 지정. 미지정 시 dufault값 password
            .defaultSuccessUrl("/design")  //직접 로그인 페이지 이동 후 성공 시 이동할 경로 지정
        }
        ```
        - 해당 경로(커스텀 로그인 페이지) 요청 처리 컨트롤러, 페이지 정의
    3. 사용자가 애플리케이션의 로그아웃 할 수 있도록 구성
        ```java
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            ...
            .and()
            .logout()  // /logout의 POST요청을 가로채는 보안 필터 설정→로그아웃 기능 제공 위해 해당 뷰에 로그아웃 폼, 버튼 추가 필요
            .logoutSuccessUrl("/");  //로그아웃 이후 이동할 페이지 지정. default는 로그인 페이지
        }
        ```
    4. CSRF 공격으로부터 보호하도록 구성
        - CSRF 공격을 막기 위해 `폼의 hidden 필드에 CSRF 토큰`을 생성하여 넣을 수 있다. 폼이 제출 될 때는 토큰과 데이터가 함께 서버로 제출되고, 서버에서는 받은 토큰을 생성한 토큰과 비교하여 일치하면 요청 처리가 허용된다.
        - Spring Security에는 내장된 CSRF 방어 기능이 기본적으로 활성화되어 있다. _csrf라는 이름의 필드를 폼에 포함시키면 된다.
        ```html
        <input type="hidden" name="_csrf" th:value="${_csrf.token}"/>
        ```
        - CSRF 비활성화는 disable()호출을 통해 가능(`비권장`) 단, REST API 서버로 실행되는 애플리케이션의 경우 CSRF disable해야한다.
#### 4.4 사용자 인지
- 사용자 인지에 사용되는 방법
    1. Principal 객체를 컨트롤러 메서드에 주입한다.
    ```java
    User user = userRepository.findByUsername(principal.getName());
    ```
    2. Authentication 객체를 컨트롤러 메서드에 주입한다.
        - 보안 특정 코드만 갖는 장점을 가진다.
    ```java
    User user = (User) authentication.getPrincipal;
    ```
    3. SecurityContextHolder를 사용해서 보안 컨텍스트를 얻는다.
        - 컨트롤러의 처리 메서드 뿐 아니라 애플리케이션의 어디서든 사용할 수 있다.
    ```java
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    User user = (User) authentication.getPrincipal();
    ```
    4. @AuthenticationPrincipal 애노테이션을 메서드에 지정한다.
        - 타입 변환이 불필요하고, 보안 특정 코드만 갖는다.
    ```java
    public String processOrder(..., @AuthenticationPrincipal User user) { ... }
    ```
> 비밀번호 암호화
>- 평문으로 저장된 비밀번호는 외부 공격으로부터 자유롭지 못하다. 
>- DB에 저장된 비밀번호, 사용자가 입력한 비밀번호는 같은 암호화 알고리즘을 사용해 암호화되어 비교한다.   
>- __암호화되어 DB에 저장된 비밀번호는 암호가 해독되지 않는다 → 단방향 암호화__
>- PasswordEncoder 인터페이스를 구현하는 클래스   
        1. BcryptPasswordEncoder : bcrypt를 해싱 암호화   
        2. NoOpPasswordEncoder : 암호화하지 않음   
        3. Pbkdf2PasswordEncoder : PBKDF2를 암호화   
        4. SCryptPasswordEncoder : scrypt를 해싱 암호화   
        5. StandardPasswordEncoder : SHA-256을 해싱 암호화   