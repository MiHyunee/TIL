## 6. REST 서비스 생성하기

#### 6.1 REST controller
- SPA (Single-Page Application, 단일-페이지 애플리케이션)
```java
@RestController
@RequestMapping(path="/design", produces="application/json")
@CrossOrigin(origins="*")
public class DesignTacoController{...}
```
- @RestController
    - 스테레오타입 애노테이션 (=이 애노테이션이 지정된 클래스를 스프링의 컴포넌트 검색으로 찾을 수 있다.)
    - 컨트롤러의 모든 HTTP 요청 처리 메서드에서 HTTP 응답 몸체에 직접 쓰는 값을 반환하겠다고 스프링에게 알려준다. (즉, 뷰를 통해 반환값이 HTML로 변환되지 않고 직접 HTTP 응답으로 브라우저에게 전달됨.)
    - 클래스에 @Controller를 사용하면 해당 클래스의 모든 요청 처리 메서드에 @ResponseBody를 지정해야 @RestController와 같은 결과를 얻을 수 있다.
- @RequestMapping
    - 속성 produces
        - 요청의 Accept 헤더에 포함된 속성의 값 요청만을 해당 컨트롤러의 메서드에서 처리함을 의미
        - 출력하고자 하는 데이터 포맷을 정의
- @CrossOrigin
    - API와 별도의 도메인(프로토콜과 호스트 및 포트로 구성)의 클라이언트에서 해당 REST API를 사용할 수 있게 해준다.
- 특정 객체만 가져오는 엔드포인트 제공하기
    - 메서드의 경로에 플레이스홀더 변수 지정하여 해당 변수를 통해 ID를 인자로 받는 메서드를 클래스에 추가
    - 메서드에서는 해당 ID를 사용해 레포지토리의 특정 객체를 찾음
    ```java
    @GetMapping("/{id}")  //{id}가 플레이스폴더
    public Taco tacoById(@PathVariable("id") Long id) {  //플레이스홀더에 대응하는 매개변수 지정
        Optional<Taco> optTaco = tacoRepo.findById(id);
        if(optTaco.isPresent()) {
            return new ResponseEntity<>(optTaco.get(), HttpStatus.OK);
        }
        return new ResponseEntity<>(null, HttpStatus.NOT_FOUND);
    }
    ```
- @PostMapping
    - HTTP POST 요청을 처리
    - 일반적으로 POST요청은 데이터 생성에 사용된다
    - 속성 consumes
        - 수신하고자 하는 데이터 포맷을 정의
        - Content-type의 값이 consumes의 값과 일치하는 요청만 처리
- @ResponseStatus
    - 해당 요청이 성공적이면서 요청의 결과로 리소스가 생성되면 전달할 상태코드 지정
    - 클라이언트에게 보다 정확한 HTTP 상태코드를 전달할 수 있다
- @RequestBody
    - 요청 몸체의 JSON 데이터가 매개변수 타입의 객체로 변환되어 매개변수와 바인딩된다는 것을 나타냄
- @PutMapping
    - HTTP 메서드 : PUT
        - 데이터를 변경하는데 사용한다
        - GET과 반대의 의미로 클라이언트로부 서버로 데이터를 전송한다. (GET 요청은 서버로부터 클라이언트로 데이터를 전송)
        - 데이터 __전체__ 교체
        ```java
        @PutMapping
        public Order putOrder(@RequestBody Order order) {
            return repo.save(order);
        }
        ```
        -  PUT은 _해당 URL에 이 데이터를 쓰라_ 의미로 생략된 속성이 있으면 해당 값을 null로 교체한다. 따라서 반드시 변경되지 않는 속성일지라도 같이 제출되어야 한다.
- @PatchMapping
    - HTTP 메서드 : PATCH
        - 데이터를 변경하는데 사용한다.
        - 데이터 __일부__ 변경
- @DeleteMapping
    - HTTP Delete 요청 처리
    - 대부분의 DELETE 요청의 응답은 body 데이터를 가지지 않으며, 반환 데이터가 없다는 것을 클라이언트가 알 수 있게 HTTP 상태 코드를 사용한다.

#### 6.2 API에 하이퍼미디어 추가하기
- REST API를 구현하는 또 다른 방법 : `HATEOAS`
    - 기존의 REST API는 API의 엔드포인트 URL이 정해지면 이를 변경하기 어렵다는 단점을 가진다. 변경되면 클라이언트 코드도 변경해야하기 때문이다.
    - HATEOAS는 API로부터 반환되는 리소스(데이터)에 해당 리소스와 관련된 하이퍼링크들이 포함되어 클라이언트가 최소한의 API URL만 알면 반환되는 리소스와 관련하여 처리 가능한 다른 API URL을 알아내어 사용할 수 있다.
    - JSON 응답에 하이퍼링크를 포함시킬 떄 주로 사용되는 형식으로 HAL(Hypertext Application Language)가 있다.
        - _links라는 속성을 포함하여 각 요소에 대한 관련 API를 수행할 수 있는 하이퍼링크를 포함한다.
- 스프링 HATEOAS
    - 의존성을 빌드에 추가
    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-hateoas</artifactId>
    </dependency>
    ```
    - 하이퍼링크 리소스를 나타내는 기본 타입
        - Resource 타입 : 단일 리소스. 다른 리소스를 링크할 수 있다.
        - Resources 타입 : 리소스 컬랙션. 다른 리소스를 링크할 수 있다.
    - 제공하는 링크 빌더
        - URL을 하드코딩하지 않고 호스트 이름을 알 수 있도록 지원
        - ControllerLinkBuilder
    ```java
    @GetMapping(/recent)
    public Resources<Resource<Taco> recentTacos() {
        PageRequest page = PageRequest.of(0, 12, Sort.by("createdAt").descending());

        List<Taco> tacos = tacoRepo.findAll(page).getContent();
        /* 방법1
        Resources<Resource<Taco>> recentResources = Resources.wrap(tacos);
        */
        List<TacoResource> tacoResources = new TacoResourcesAssembler().toResources(tacos);
        Resources<TacoResource> recentResources = new Resources<TacoResoure>(tacoResources);

        recnetResources.add(
            /* 방법1
            new Link("http://localhost:8080/design/recent", "recents")  //이름이 recents, 해당 url인 링크를 추가 
            */
            /* 방법2
            ControllerLinkBuilder.linkTo
            (DesignTacoController.class)  //호스트이름과 기본경로
            .slash("recent")  //경로 추가
            .withRel("recents")  //링크의 관계 이름 지정
            */
            //방법3
            ControllerLinkBuilder.linkTo(methodOn(DesignTacoController.class).recentTacos())
            .withRel("recents")
        );
        return recentResources;
    }
    ```
    ```JSON
    "_links": {
        "recents": {
            "href": "http://localhost:8080/design/recent"
        }
    }
    ```
    - 리소스 어셈블러 생성
        - 방법1 : 반복 루프에서 Resources객체가 가지는 각 Resouce<T>요소에 Link 추가 (비권장)
        - 방법2 : T객체를 Resources.wrap()에서 Resource객체로 생성하지 않고, T객체를 새로운 TResource 객체로 변환하는 유틸리티 클래스 정의하여 링크를 갖는 필드 추가
            > @Relation : java로 정의된 Resource 타입 클래스 이름과 JSON 필드 간의 결합도 낮추는 방법 
            > JSON필드 이름을 짓는 방법을 지정
            > ```java
            > @Relation(value="taco", collectionRelation="tacos")
            > public class TacoResource extends ResourceSupport {...}
            >```
            > ```JSON
            > {
            >    "_embedded": {
            >      "tacos": [...]
            >   }
            > } 
            >```
            > - @Relation지정되지 않았다면 List<TacoResource>로 부터 생성되어 "tacoResourceList"로 표기되었을 것이다.
            ```java
            @Getter
            public class TacoResource extends ResourceSupport {
                private final String name;
                private final Date createdAt;
                private final List<Ingredient> ingredients;
                
                public TacoResource(Taco taco) {
                    this.name = taco.getName();
                    this.createdAt = taco.getCreatedName();
                    this.ingredients = taco.getIngredients();
                }
            }
            ```
            - ResourceSupport의 서브클래스로 Link객체 리스트와 이를 관리하는 메서드를 상속받는다
            - DB에서 필요한 id를 API에 노출시킬 필요가 없어 객체(Taco)의 id 속성을 갖지 않는다.
            - API클라이언트 입장에서 해당 리소스의 self 링크가 리소스 식별자 역할을 한다.
            ```java
            //리스트의 Taco 객체를 TacoResouce객체들로 변환하기 위한 리소스 어셈블러 클래스
            public class TacoResourceAssembler extends ResouceAssemblerSupport<Taco, TacoResource> {
                public TacoResourceAssembler() {
                    super(DesignTacoController.class, TacoResource.class);
                }  //TacoResource를 생성하면서 만들어지는 링크에 포함되는 url의 기본 경로를 결정하기 위해 DesignTacoController사용

                @Override
                protected TacoResource instantiateResource(Taco taco) {
                    return new TacoResource(taco);
                }  //TacoResource가 기본생성자 갖는다면 생략 가능

                @Override
                protected TacoResource toResource(Taco taco) {
                    return createResourceWithId(taco.getId(), taco);
                }  //반드시 오버라이드 되어야한다.
            }
            ```
            - instantiateResource()는 Resource 인스턴스만 생성, toResource()는 Resource인스턴스를 생성하면서 링크도 추가
            - toResource()는 내부적으로 instantiateResource()를 호출
- 데이터 기반 서비스 활성화 : 데이터 리퍼지터리 기반으로 스프링 데이터 REST가 API 자동 생성
    - 스프링 데이터 REST
        - 스프링 데이터의 또 다른 모듈로 스프링 데이터가 생성하는 리퍼지터리의 REST API를 자동생성
        - 빌드 추가
        ```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-rest</artifactId>
        </dependency>
        ```
        - 스프링 데이터 REST가 자동으로 생성하는 엔드포인트를 사용하기 위해서는 @RestController이 지정되어있던 모든 클래스를 제거해야 한다.
        - 엔드포인트는 GET, POST, PUT, DELETE 메서드 지원
        - 자동 생성된 API와 관련해 해야 할 작업은 __해당 API의 기본 경로를 설정하는 것__
        ```yml
        spring:
            data:
                rest:
                    base-path: /api  #스프링 데이터 REST 엔드포인트의 기본경로 /api
        ```
        - 리소스 경로와 관계 이름 조정
            - 엔드포인트를 생성할 때 관련 엔티티 클래스 이름의 복수형 사용(Order → /orders, Taco → /tacoes)
            - 엔티티 클래스에 @RestResources 지정하면 관계 이름, 경로 지정 가능 
            ```java
            @Data
            @Entity
            @RestResource(rel="tacos", path="tacos")
            public class Taco {...}
            ```
        - 페이징, 정렬
            - 컬렉션 리소스를 요청하면 한 페이지 당 20개 항목이 기본값.
            - first, last, next, previous 페이지 링크를 요청 응답에 제공
    - 스프링 데이터 REST : 커스텀 엔드포인트 추가
        - @RestController가 지정된 빈을 구현하여 자동 생성 엔드포인트에 보충
        - 주의점
        > 1. 스프링 데이터 REST의 기본 경로를 포함하여 우리가 원하는 경로가 앞에 붙도록 매핑필요. 기본 경로 변경될 시 해당 컨트롤러의 매핑 역시 수정    
        >       - 스프링 데이터 REST는 @RespositoryRestController를 포함
                - 스프링 데이터 REST 엔드포인트에 구성되는 기본 경로와 동일하게 매핑되는 컨트롤러 클래스에 지정하는 새로운 애노테이션.
                - 포함되는 메서드에 @ResponseBody 지정하거나 응답 데이터를 포함하는 ResponseEntity반환해야한다. (핸들러 메서드의 반환값을 요청 응답의 몸체에 자동으로 수록하지 않음)
        > 2. 커스텀한 엔드포인트는 스프링 데이터 REST 엔드포인트에서 반환되는 __리소스의 하이퍼링크에 자동으로 포함되지 않음__. 즉, 클라이언트가 관계 이름을 사용해 커스텀 엔드포인트를 찾을 수 없음을 의미.
        >       - __리소스 프로세서 빈__ 선언하여 링크 추가
        >       - 엔드 포인트의 반환 타입의 리소스에 링크를 추가하는 ResourceProcessor 구현
        > ```java
        > @Bean
        > public ResourceProcessor<PagedResources<Resource<Taco>>> tacoProcessor(EntityLinks links) {
        >    return new ResourceProcessor<PagedResources<Resource<Taco>>>() {
        >        @Override
        >        public PagedResources<Resource<Taco>> process(PagedResources<Resource<Taco>> resource) {
        >            resource.add(links.linkFor(Taco.class).slash("recent").withRel("recents"));
        >            return resource;
        >        }
        >    }
        >}
        >```
    