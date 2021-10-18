[G&B 다이렉트 오더(크롤링) API 연동](https://www.notion.so/G-B-API-082f52fab51244bf8ae55801b5f38d76)

## 미정인 정책 (결정에 따라 추가 개발)

1. G&B에서 응답한 주문 번호 관리 방안
    1. 2.0에서 테스트 이후 결정
    2. 결정 사항에 따라서 Order-Creator와 drf에 추가 개발 가능성
2. color 필드에 A1192, F0UU0 같은 코드 명(?)이 들어오는 경우 처리
    1. 결정될 방안에 따라 크롤러와 프론트엔드까지 영향력이 있을 수 있음
3. 가격 산정 방식은 확정이 된 것인지?
    1. 부티크 별, 브랜드 별, 개별 상품 별 등등...의 가능성
4. G&B 상품 이미지의 http 링크 이슈
    1. [추가] G&B에서 제공할 수 있는 방안 없음
    2. [추가] 이미지를 디코드 S3에 백업할 경우 전체 상품 크롤링 시간의 증가 예상
5. (상품이 삭제돼서 자동 품절 취소가 된 뒤, 고객에게 공지가 가지 않으면 CX 이슈가 예상되는데... 이때 추가 개발 요청이 들어올 가능성도...)
6. 다이렉트 오더 구매 동의 팝업
    - 구매동의 API를 통해 등록여부에 따라 구매진행
    - 구매동의 API 실패 시 구매불가
    - 구매동의 실패 시 에러메시지 서버에서 전달
7. 프리오더, 다이렉트 상품을 장바구니에서 합주문 할때 동의 팝업을 여러개 띄워야 하는지?

## [pycrawler] 개발 예정 항목

1. [추가] 크롤링할 때 상품의 색상 컬럼 수정
    1. CSV 파일 검토 결과 색상을 나타내는 컬럼이 여러 개 있어서 이 중에서 가장 적합한 것을 선택하도록 변경
2. 상품 > 색상, 크기의 계층 형태로 상품 크롤링
    1. sku 값으로 Modello를 사용하도록 변경, 기존 sku였던 CodArticolo는 별도 컬럼이나 테이블로 관리
        1. CodArticolo는 특정 모델명/색상/크기 조합을 나타내는 일종의 키 값임 (= 주문 필수 값)
    2. 현재 상품 등록 시 크롤러에서 drf로 API 호출을 통해 하고 있으므로, drf에 새로운 API를 구현하거나 크롤러에서 CodArticolo 등록을 따로 처리해야 함
3. 현재 주문 상태(해외 주문 확인 중 - 디코드 주문 상태 / 해외 주문 완료 - G&B 주문 완료)에 따라 크롤링 상품의 재고 변경
    1. 크롤러에서 주문 정보를 조회할 수 있어야 함 (자체적으로 처리 / API 사용 고려)
4. 신규 브랜드와 카테고리 조합 등록 후 상품 업데이트
    1. 전체 크롤링을 한 번 돌려야 하는데, 지금은 개발자가 직접 젠킨스 배포를 통해서만 크롤러를 임의 실행할 수 있는 상태
    2. 이 경우 기존에 이미 조합된 상품은 건너뛰면서 크롤링해야 빠르게 상품 정보를 업데이트할 수 있으므로 크롤러의 로직을 분기하는 것도 고려할 필요가 있음 (전체 크롤링을 하면 3시간이 걸리기 때문에...)
    3. [추가] 신규 매핑한 상품을 즉각 반영해야 한다면, 매핑 완료 후 크롤러를 실행하려고 할 때 진행 중인 크롤러가 있을 수 있음. 이 경우 큐를 이용하거나 진행 중인 크롤링 작업을 종료 후 다시 실행하거나 하는 등의 분기 처리와 배치 잡 컨트롤이 필요함
    4. [추가] 하지만 주 1회 매핑 전략임을 감안해서, 다음 날 첫 크롤링 작업 때 매핑한 상품이 반영된다고 정책을 세우면 추가 개발 요소는 없을 것으로 판단
5. 삭제된 상품 처리 (피드 미노출, 자동 품절 취소)
    1. 전체 상품 크롤링(하루의 첫번째)을 마친 후 업데이트 되지 않은 상품은 피드 미노출 처리
    2. [수정] 하루의 첫 크롤링이 끝난 뒤 바로 실행하려면 pycrawler에서 하는게 좋겠지만, 삭제된 상품의 주문 취소도 동시에 처리해야 빠를테니 Order-Creator에서 일괄 담당하는게 나을 것으로 판단했으나... pycrawler와 Order-Creator에서 각각 병렬 처리하는 것도 고려
    3. 어느 서버에서 처리하든 pycrawler에서 크롤링 종료후 API 요청을 보내서 처리하면 될 듯
6. 15분 마다 크롤링
    1. 지금은 크롤러가 하루의 모든 CSV를 읽으므로, 상황에 따라 특정 파일만 크롤링하도록 수정 필요 (삭제된 상품 처리와도 연관)
    2. ECS 태스크에서 배치 설정 변경 (또는 애플리케이션 단계에서 배치 컨트롤...)
    3. [추가] ECS 태스크의 cron 작동 방식에 대해서 채철님에게 문의 결과, 전체 상품 크롤링을 마친 다음부터 15분 마다 크롤링을 실행하도록 동적인 작업을 하려면 pycrawler와 ECS가 연동되도록 추가 작업 또는 인프라가 필요함
    4. [추가] 손쉽게 하려면 전체 상품 크롤링 시작 후 X시간 뒤부터 15분 마다 크롤링하도록 태스크를 2개 만드는 것. 하지만 전체 상품 크롤링이 무조건 X시간 안에 끝난다는 보장이 있어야 하므로 위험함

## [Order-Creator] 개발 예정 항목

1. [추가] 주문 요청 객체의 생성 방식 변경
    1. 기존 product 테이블의 sku가 아닌 새로운 주문 필수 값(위 pycrawler 1번 참조)을 사용하도록 변경
2. 삭제된 상품 처리 (피드 미노출, 자동 품절 취소)
    1. `[pycrawler] 개발 예정 항목` 4번과 동일
    2. 전체 상품 크롤링을 마친 후 업데이트 되지 않은 상품의 주문을 취소 처리
3. G&B 주문 생성이 실패했을 때의 디코드 주문 데이터 저장과 오류 로깅
4. 슬랙 연동?

## [DRF] 개발 예정 항목

1. 브랜드와 카테고리 매핑 완료 후 크롤러 실행 요청

## [DB] DDL

- 매 시점마다 한 번만 G&B 주문 생성을 요청해야 하므로 shedlock을 사용해서 여러 Order-Creator 서비스 중 한 대만 실행하도록 제한
    
    ```sql
    CREATE TABLE shedlock (
    	name varchar(64) NOT NULL,
    	lock_until timestamp(3) NULL DEFAULT NULL,
    	locked_at timestamp(3) NULL DEFAULT NULL,
    	locked_by varchar(255) DEFAULT NULL,
    	PRIMARY KEY (name)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8
    ```
    
- 모델명 / 색상 / 크기 조합에 해당하는 키 값(variant_code) 저장
    
    ```sql
    CREATE TABLE `product_stock_gnb` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT,
      `product_stock_id` bigint(20) NOT NULL,
      `variant_code` varchar(60) NOT NULL,
      PRIMARY KEY (`id`),
      UNIQUE KEY `unique_product_stock_gnb_product_stock_id` (`product_stock_id`),
      CONSTRAINT `fk_product_stock_gnb_product_stock_id` FOREIGN KEY (`product_stock_id`) REFERENCES `product_stock` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
    );
    ```