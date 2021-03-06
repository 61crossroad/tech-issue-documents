## 요약

- 커버 서버에서 에러가 발생해서 요청량을 처리하지 못하고 다운 됨

## 영향

- web, app 모든 서비스에서 커버, 상품 표시 되지 않아서 정상적인 서비스가 불가능
- 서비스 장애 시간: 11:00 ~ 11:10

## 원인

AWS/클라우드 워치와 엘라스틱 서치/APM의 기록을 보면 크게 3개 로그로 정리할 수 있습니다.

1. could not initialize proxy [dcode.model.persistence.entity.content.ContentBox#1106] - no Session \
dcode.model.persistence.entity.content.ContentBox_$$*jvst84c_7.getBoxData(ContentBox*$$_jvst84c_7.java)
2. Unable to acquire JDBC Connection; nested exception is org.hibernate.exception.JDBCConnectionException: Unable to acquire JDBC Connection
3. WARN ObjectGraphWalker - The configured limit of 100,000 object references was reached
while attempting to calculate the size of the object graph. 
Severe performance degradation could occur if the sizing operation continues. 
This can be avoided by setting the CacheManger or Cache <sizeOfPolicy> elements maxDepthExceededBehavior to "abort" or adding stop points with @IgnoreSizeOf annotations. 
If performance degradation is NOT an issue at the configured limit, raise the limit value using the CacheManager or Cache <sizeOfPolicy> elements maxDepth attribute. 
For more information, see the Ehcache configuration documentation.

1이 가장 주요한 원인인데, CoverCollectionService#getCollection에서 발생한 익셉션으로서 디스커버 상세를 조회하는 로직입니다. 해당 메서드의 절차를 발췌하면...

1. AtomicCoverCollectionService#getCoverCollection**(ehcache 적용)**을 호출해서 CoverCollection 엔티티를 얻습니다. **(엔티티 캐싱 발생)**
2. CoverCollection이 필드로 가지고 있는 ContentBox 엔티티의 BoxData를 가져옵니다. **(캐시 되지 않음)**
3. ... 생략 ...

기존에는 CoverCollection.ContentBox의 페치 타입이 EAGER였기때문에 2에서 ContentBox와 BoxData를 바로 가져올 수 있지만 LAZY로 바뀌면 문제가 발생합니다.

이를테면, id=9999인 CoverCollection을 조회하려는데 ehcache가 아직 이 엔티티를 캐싱하지 않았다면 아무 문제없이 조회를 완료하고 ehcache는 9999를 캐싱합니다. 
하지만 캐싱된 9999의 ContentBox에는 실제 인스턴스가 아닌 프록시 객체가 들어있습니다. 
그렇기 때문에 다음부터 CoverCollection(id=9999)를 조회하면 캐싱된 객체를 받게되고, BoxData를 받기 위해 getContentBox()를 호출하면 실제 객체가 아닌 프록시가 나타나버립니다.

사실, 기존 소스에 @Transactional이 명시되어 있지 않아서 정확한 트랜잭션 범위(=영속성 컨텍스트 존재 여부, OSIV를 사용하지 않았으므로 
트랜잭션 범위와 영속성 컨텍스트 존재 범위가 같다고 생각했습니다)를 알기는 어렵습니다만... 
ContentBox에는 이전에 캐싱된 프록시 객체가 있으므로 Jpa는 프록시를 따라서 실제 객체를 조회하려고 시도합니다.

하지만 ContentBox에 할당된 프록시는 이미 이전에 조회가 끝난 프록시였고, 다른 쓰레드(=다른 영속성 컨텍스트)에서 생성된 프록시였기 때문에 
could not initialize proxy ~ no Session, LazyInitializationException이 발생합니다. 
(로그 1을 다시 보면 ContentBox의 이름 뒤에도 역시나 프록시를 뜻하는 *$$*jvst...가 붙어있습니다)

LazyInitializationException이 발생할 경우 하이버네이트가 어떻게 대처하거나 동작하는지 알고 싶었지만 이 부분을 설명한 문서를 찾지 못했고, 
테스트를 해보기에도 오랜 시간이 걸려서 아직 밝혀내지는 못했습니다. 
다만, 일반적으로 DB 관련 동작은 꽤 긴 시간 동안 커넥션을 유지하며 여러 번 시도를 하기에(흔한 경우로 연결되지 않은 DB에 접속을 시도하면 꽤 오랫동안 로딩을 하며 access를 하려고 하는 등...)
이 역시 비슷했을 것이라고 예상합니다. 
즉, 본 익셉션으로 인해 JDBC 커넥션을 무리하게 오래 유지했을 가능성이 크며 그로 인해 2번의 Unable to acquire JDBC Connection이 발생했을 것입니다. 
(이럴 경우 따라오는 java.sql.SQLTransientConnectionException: SpringBootJPAHikariCP - Connection is not available, request timed out after 3000Xms. 로그도 상당수 발견했습니다.)

3번, WARN ObjectGraphWalker 로그는 엔티티의 proxy 객체까지 전부 캐싱하려다가 엔티티 그래프 끝까지 탐색하게 되고 길을 잃어서 발생했습니다. 
(구글링해보면 실제로 이런 이슈가 좀 나옵니다) 해당 로그가 발생한 부분 역시 CoverCollectionService#getCollection 메서드이며 전체 쿼리 로그를 요약하면,

1. CoverCollection 조회
2. WARN ObjectGraphWalker - The configured... 발생
3. ContentBox 조회
4. AtomicCoverCollectionLikedService - ------------>>>> getCollectionLiked
5. AtomicCoverCollectionAwaiterService - ------------>>>> getCollectionAwaiter
6. AtomicCoverCommentService - ------------>>>> getAllCoverCommentsCount

로 정리됩니다.

이 중 CoverCollectionAwater나 CoverCollectionLiked 들도 엔티티이며, 해당 AtomicService의 메서드에 캐싱이 걸려있지만 연관된(=필드로 갖고 있는)엔티티가 없어서 엔티티 그래프를 탐색하지 않고, 
WARNING도 발생하지 않았습니다. 정리하자면 엔티티는 Jpa L2 Cache로 캐싱해야 합니다. (로그 내용처럼 이 캐싱 이슈도 서버의 성능 저하에 영향을 미쳤겠지만, 구체적인 영향력은 미지수입니다)

이렇게 주요한 로그에 대해 파악을 했고, 캐시 사용에 대해 조금 더 적어보려고 합니다. 
스프링 애플리케이션에서 보편적으로 사용하는 캐싱은 각 요청 쓰레드에 상관없이 마치 전역 변수처럼 설정되어 있습니다. 
그러므로 이런 캐싱을 개별 쓰레드에 한정되어 있는 트랜잭션과 영속성 컨텍스트 및 엔티티에 적용하면 이번 사례와 같이 예상치 못한 문제가 발생할 수 있습니다. 
Jpa 엔티티 관련 작업은 오직 명시적 트랜잭션과 영속성 컨텍스트 라이프 사이클 안에서만 제대로 작동하기 때문입니다.

그렇다 하더라도 Jpa의 엔티티에 대해 캐싱이 불가능하지 않으며, 
오히려 Jpa에서 제공하는 L2 Cache를 도입하면 캐시 레이어를 2단계로 나눌 수 있어서 견고하고 효율적인 캐싱 아키텍처를 만들 수 있을 것이라고 생각합니다.

Controller ← DTO 캐시 객체 ← Service ← 엔티티 2차 캐시 객체 ← Repository - JDBC I/O

위처럼 엔티티 캐싱은 영속성 컨텍스트와 JDBC 사이의 Jpa L2 Cache가 전담하도록 하고, 
조회된 엔티티를 가공해 만든 DTO는 기존 서비스 레이어에 설정한 캐싱을 그대로 사용할 수 있습니다.

소프트웨어 아키텍처 관점에서 이 부분을 보자면, 비즈니스 레이어와 DB 레이어가 완전히 격리되지 않은 것도 원인이라고도 말할 수 있을 것 같습니다. 
비즈니스 레이어에서 구현한 캐싱이 DB 레이어에도 영향을 미쳤으니까요...

## 계기

이 이슈는 N + 1 쿼리를 해결하기 위해 CoverCollection 엔티티의 연관 엔티티 - 글로벌 페치 전략을 EAGER → LAZY로 변경했다가 발생했습니다.

거슬러 올라가면, 프리뷰를 구현하기 위해서 기존에 있던 로직을 파악했고 Junit으로 CoverCollectionService#getAllCollection을 테스트했습니다. 
그 결과 Hibernate 로그에서 N + 1 쿼리를 발견했고, 해결하고 넘어가야할 문제라고 판단을 하게 되었습니다.

N + 1이 발생한 근본적인 원인을 다시 살펴보면
DB의 cover_collection 테이블에서 content_box_id, product_box_id, bottom_content_box_id 컬럼에 참조 무결성을 위배하는 -1이라는 값이 FK로 할당되었기 때문입니다.

*** 2.0의 DB 스키마는 전반적으로 FK 값이 없을 경우 -1로 설정하는 전략을 사용합니다. 
예를 들어 brand 테이블은 id = -1인 row에 NOT MATCHED라는 값으로 null 참조를 처리하고 있습니다. 
하지만 content_box와 product_box 테이블은 PK 컬럼이 unsigned로 선언이 되어 있어서 id = -1인 row를 생성할 수 없고, 결국 존재하지 않는 값을 참조하게 되었습니다.

물론 cover_collection 테이블을 살펴본 결과 어느 시점부터 content_box_id와 product_box_id에는 -1이 아닌 null이 할당되었고, 
drf-santamonica에서도 콘텐츠 박스와 상품 박스 입력시에 null로 insert하도록 이미 조치가 되어 있었습니다. 
(현재 bottom_content_box_id에 대한 null 처리도 반영했습니다)

그러나 CoverCollection 엔티티에는 -1로 인한 참조 무결성 문제를 우회하기 위한 @NotFound(action = NotFoundAction.IGNORE) 애노테이션이 여전히 적용되어 있었습니다. 
이 애노테이션은 재밌는 것이... 참조를 찾지 못한 JoinColumn을 단순히 무시하고 지나가는게 아니라, IGNORE 상태를 만들기 위해 무조건 EAGER로 페치 해버리는 숨겨진 기능이 있습니다. 
(그래서 글로벌 페치 전략을 LAZY로 바꾸더라도 NotFound 애노테이션이 있으면 무조건 EAGER로 페치합니다)

여기까지 파악한 후 제가 취한 방법은

1. cover_collection 테이블에서 -1인 FK를 모두 null로 변경
2. @NotFound 애노테이션 제거
3. 글로벌 페치 전략을 LAZY로 변경

이었습니다. 이 결과 getAllCollection의 목록 조회에서 N + 1은 없앨 수 있었습니다.

다만, getCollection 메서드가 디스커버를 상세 조회할 때 발생한 엔티티 프록시와 캐싱의 충돌까지는 예상하지 못했습니다.

## 해결

- 4월 27일 이전 버전으로 롤백해서 해결

## 추후 방안

- **[DAO 아키텍처]** 현재 *** 2.0과 drf-santamonica의 ORM에서 N + 1 쿼리가 꽤 존재하는 것으로 파악했습니다. 
페치 전략과 쿼리를 수정하거나 ORM을 아예 제거해서 근본적으로 해결할 필요가 있습니다. 
어떤 방식을 택해야 하느냐에 정답은 없겠지만, 개인적으로는 Jpa를 Best Practice로 리팩토링하는 쪽에 조금 더 마음이 갑니다. 
ORM이라는 패러다임이 러닝 커브가 높고 까다로운 기술임에는 분명합니다. 
그러나 그 고점을 넘기만 한다면 다른 DAO 기술보다 빠르게 CRUD를 쳐낼 수 있을 것 같습니다. 
또한 Jpa는 JdbcTemplate과 같은 수동 매핑, MyBatis같은 xml 매핑 방식까지 모두 제공하기 때문에 유연한 DAO 아키텍처를 구현하기에도 좋다는 생각입니다.

- **[캐싱]** 캐싱에 관해서는... N + 1 쿼리 로직에 이미 캐싱이 잘 걸려 있기때문에 현재 성능상 큰 이슈는 없어 보입니다. 
그러니 엔티티에 걸린 캐싱 부분만 걷어내고 해당 엔티티를 가공한 DTO만 캐싱하거나, 엔티티에는 Jpa L2 Cache를 따로 적용하는 방식도 고려해볼만 합니다.

- **[테스트]** 사실 도메인의 중심 엔티티에 설정된 글로벌 페치 전략을 바꾸는 것이 리스크가 큰 작업이라는 것은 자각하고 있었습니다.
그래서 나름대로 예외처리를 위한 테스트도 진행했지만 Covers MSA에 대해 전체적인 파악도 되지 않은 상태였기에 무리한 시도였다는 생각이 듭니다.
다만, 이런 경우에 대비해서 각 Service 클래스 단위, 또는 도메인 단위로 모든 기능을 테스트하는 '통합 테스트 코드'를 마련해두는게 어떨까하는 생각을 합니다. 
통합 테스트가 있다면 큰 규모의 리팩토링에서 발생할 수 있는 예상치 못한 오류를 찾아내기도 쉬울 것이며, 
특정 기능의 성능 최적화나 유지보수에서 버그, 기능 수정 시에도 사이드 이펙트를 최소화 할 수 있을 것 같습니다.
- 나아가 트래픽 테스트 시스템도 정리해보면 좋겠습니다.
