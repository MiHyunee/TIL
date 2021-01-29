## 5. Working with Configuration properties

#### 5.1 Auto-configuration 세부 조정하기
- 스프링 환경 추상화
    - 구성 가능한 모든 속성을 한 곳에서 관리한다는 개념
    - 속성의 근원을 추상화하여 속성을 필요로 하는 bean이 스프링 자체에서 해당 속성을 사용할 수 있게 해줌
    - `JVM 시스템 속성, 운영체제의 환경 변수, 명령행 인자, 애플리케이션의 속성 구성 파일(applicaiton.properties파일, application.yml파일)`의 원천 속성을 스프링 환경에 가져온다.
    - 스프링 부트에 의해 자동으로 구성되는 빈들은 스프링 환경으로부터 가져온 속성들을 사용해 구성될 수 있다.
- 구성 속성 = 빈의 속성
- Spring의 구성
    1. Bean 연결
        - Spring Application Context에서 bean으로 생성되는 애플리케이션 컴포넌트 및 상호 간에 주입되는 방법을 선언하는 구성
    2. 속성 주입
        - Spring Application Context에서 bean의 속성 값을 설정하는 구성
- DataSource 구성
    - DataSource 빈을 명시적으로 구성 가능.
    - 스프링 부트에서는 명시적 구성 불필요하며 구성 속성을 통해 간단하게 가능.
    ```yml
    #applicaiton.yml
    spring:
        datasource:
            url: jdbc:mysql://localhost/tacocloud
            username: tacodb
            password: tacopassword
    ```
- 내장 서버 구성
    - 서블릿 컨테이너의 포트 설정하기
        - 포트 0번으로 명시하면 사용 가능한 포트를 무작위로 선택하여 시작한다. __자동화된 통합 테스트__ 실행 시 포트가 충돌하지 않고, __마이크로서비스는__ 포트가 중요하지 않아 유용하다.
    ```yml
    #application.yml
    server:
        port: 0 
    ```
    - HTTPS 요청 처리 위한 컨테이너 관련 설정
        1. JDK의 keytool 명령행 유틸리티를 사용해서 keystore 생성
        ```cmd
        $ keytool -keystore mykeys.jks -genkey -alias tomcat keyalg RSA
        ```
        2. application.properties/application.yml에 속성 설정
        ```yml
        server:
            port: 8443  #개발용 HTTPS 서버에 많이 사용
            ssl: 
                key-store: file:///path/to/mykeys.jks  #키스토어 파일이 생성된 경로. 애플리케이션의 JAR 파일에 키스토어 파일 넣는 경우 classpath:를 URL로 지정
                key-store-password: letmein  #키스토어 생성 시 지정한 비밀번호
                key-password: letmein  #키스토어 생성 시 지정한 비밀번호
        ```
- 로깅 구성
    - 스프링 부트는 콘솔에 로그 메시지를 쓰기 위해 Logback을 사용해 로깅을 구성한다. 
    - logback.xml 파일을 수정하면 원하는 형태로 애플리케이션 로그 파일 제어 가능하지만 스프링 부트에서는 구성 속성을 사용하여 logback.xml 파일 생성 없이 변경할 수 있다.
    - 로깅 구성에서 많이 변경하는 내용
        1. 로깅 수준
        2. 로그 수록할 파일
    ```yml
    #application.yml
    logging:
        path: /var/logs/   #경로
        file: TacoCloud.log  #수록 파일 지정
        level:
            root: WARN
            org.springframework.security: DEBUG  #가독성 위해 한 줄로 지정
    ```
    - 기본 로그파일 크기(10MB)가 차면 새로운 로그 파일 생성되어 로그 항목 계속 수록된다. 스프링 2.0부터는 날짜별로 로그 파일이 남으며 지정된 일 수가 지난 로그 파일은 삭제된다.
- __다른 구성 속성으로부터 값 가져오기__
    - ${  }사용해서 값 가져올 수 있다

#### 5.2 구성 속성 생성하기
- __@ConfigurationProperties__ 애노테이션을 지정하면 해당 빈의 속성들이 스프링 환경의 속성으로부터 주입된다.
```java
...
@Controller
@ConfigurationProperties(prefix="taco.orders")  //구성 속성 값 설정 시 taco.orders.pageSize라는 이름 사용
public class OrderController {
    private int pageSize = 20;  //속성 기본 값=20

    public void setPageSize(int pageSize) {
        this.pageSize = pageSize;
    }

    @GetMapping
    public String ordersForUser(@AuthenticationPrincipal User user, Model model) {
        Pageable pageable = PageRequest.of(0, pageSize);
        model.addAttribute("orders", orderRepo.findByUserOrderByPlacedAtDesc(user, pageable));
    }
}
```
1. application.yml을 통해 구성 속성 설정하기
```yml
taco:
    orders:
        pageSize: 10
```
2. 환경 변수로 구성 속성 설정하기
    - 애플리케이션을 프로덕션에서 사용 중에 빨리 변경할 때 유용
```
$ export TACO_ORDERS_PAGESIZE=10
```
- 구성 데이터를 속성 홀더(property holder)에 설정하기
    - @ConfigurationProperties는 구성 데이터의 holder로 사용되는 빈에 지정되는 경우가 많다.
        - 장점 : 외부에 구성 관련 정볼르 따로 유지할 수 있음, 여러 빈에 공통적인 구성 속성을 쉽게 공유 가능
    ```java
    /*
    OrderController의 pageSize 속성을 별개의 holder class로 분리
    */
    @Component
    @ConfigurationProperties(prefix="taco.orders")
    @Data
    public class OrderProps {
        private int pageSize = 20;  //속성 기본값 = 20
    }
    ```
    - 구성 속성 홀더는 스프링 환경으로부터 주입되는 속성들을 갖는 빈. 즉, 해당 속성들이 필요한 다른 빈에 주입될 수 있다. 
    - 구성 속성 홀더를 통해 코드 재사용에 용이하고, 속성의 추가, 삭제, 수정 등이 편리하다.
- 구성 속성 메타데이터 선언
    - 메타데이터는 선택적. 없더라도 구성 속성이 동작하는데 영향주지 않는다. 다만, 있으면 속성의 정보를 최소한으로 자세히 보여준다. 
    - spring-boot-configuration-processor 의존성 추가(@ConfigurationProperties 애노테이션이 지정된 애플리케이션 클래스에 대한 메타데이터 생성하는 애노테이션 처리기)
    - src/main/resource/META-INF 아래에 additional-spring-configuration-metadata.json 파일 생성하여 메타데이터 생성
    ```json
    {
        "properties" : [
            {
                "name" : "taco.orders.page-size",
                "type" : "int",
                "description" : "Sets the maximum number of orders to display in a list."
            }
        ]
    }
    ```
    ###### _스프링 부트는 속성 이름을 유연하게 처리하므로 taco.orders.page-size와 taco.orders.pageSize를 같은 것으로 간주_

#### 5.3 profile
- 서로 다른 설치 환경에 서로 다른 속성을 구성해야 할 때
    1. 환경 변수로 데이터베이스 구성 속성을 설정한다.
        - 여러 구성 속성을 환경 변수로 지정하는 것은 번거롭다.
        - 환경 변수의 변경을 추적 관리하거나 오류가 있을 경우 변경 전으로 되돌리기 어렵다.
    2. __profile을 사용하여 설정한다.__
        - 런타임 시에 활성화되는 profile에 따라 서로 다른 빈, 구성 클래스, 구성 속성들이 적용 또는 무시되도록 한다.
        1. application-[프로파일 이름].yml 또는 application-[프로파일 이름].properties 파일 이름 규칡을 따른 환경에 해당하는 속성들만 포함되는 파일을 별도로 생성한다.
        2. _(YAML 구성에서만 가능)_ 프로파일에 특정되지 않고 공통으로 적용되는 기본 속성과 함께 프로파일 특정 속성을 application.yml에 지정 가능
        ```yml
        #공통으로 적용되는 속성
        logging:
            level:
                tacos: DEBUG
        ---   #구분선
        spring:
            profiles: prod  #prod profile에만 적용됨
            datasource: 
                url: jdbc:mysql://localhost/tacocloud
                username: tocouser
                password: tacopassword

        logging:
            level:
                tacos: WARN
        ```
        - profile 활성화하기
            - profile 특정 속성들의 설정은 해당 profile이 활성화되어야 유효하다
            - active속성에는 여러 개의 profile이 포함될 수 있다.
            1. application.yml에 spring.profiles.active 속성에 지정 (비권장)
                - application.yml에 활성화 profile을 설정하면 해당 profile이 기본 profile이 된다. → 프로덕션 환경 속성과 개발 속성 분리의 장점을 살릴 수 없다
            ```yml
            spring:
                profiles:
                    active:
                    -prod
                    -audit
            ```
            2. 환경 변수를 사용해서 활성화 profile 설정
                - 해당 컴퓨터에 배포되는 어떤 애플리케이션에서도 prod profile이 활성화된다. 기본 profile의 동일한 속성보다 더 높은 우선순위를 갖는다.
            ```
            % export SPRING_PROFILES_ACTIVE=prod
            ```
            3. 명령행 인자로 활성화 profile 설정
            ```
            java -jar taco-cloud.jar --spring.profiles.active=prod, audit
            ```
- profile을 사용해서 조건별로 빈 생성하기
    - 일반적으로 자바 구성 클래스에 선언된 빈은 활성화되는 profile과 무관하게 생성된다
    - @Profile 애노테이션을 사용하면 지정된 profile에만 적합한 빈들을 나타낼 수 있다.
