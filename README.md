# 4주차 미션 리뷰 기반 설계 개선

## 1. 엔티티별 계층 분리 → Aggregate 단위 관리
### 문제

`Item`에 옵션 기능을 추가하며 Option 엔티티에 대해서도 `OptionRepository, OptionService, OptionController`를 각각 생성했습니다. 엔티티가 늘어날 때마다 동일한 계층을 반복 생성하는 구조였고, 옵션 조회·추가·삭제가 `Item`을 거치지 않고 독립적으로 이뤄질 수 있었습니다.

### 리뷰

> 모든 엔티티마다 Repository, Service를 만들어서 값을 리턴해야 하는 건 아니에요.
Product와 Option은 생명주기가 같기 때문에 옵션에 대한 값을 리턴할 때는 ProductRepository를 통해서 값을 리턴해보시는 걸 권장드려요.

### 판단

`Option`은 `Item` 없이는 존재할 수 없고, 생성·삭제 시점이 `Item`에 종속됩니다. 즉 독립적인 조회·수정 진입점을 가질 이유가 없는 엔티티였습니다. `Item`을 `Aggregate Root`로 두고 옵션에 대한 모든 접근을 `Item`을 통해서만 하도록 경로를 단일화했습니다.

### 적용
`OptionRepository, OptionService` 및 관련 테스트 제거
옵션 관련 비즈니스 로직을 `ItemService`로 통합
`@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)` 적용
옵션 추가: `Item`의 컬렉션에 추가 후 Item 저장
옵션 삭제: 컬렉션에서 제거하면 고아 객체로 함께 삭제
결과

옵션의 생명주기가 Item에 의해서만 관리되어, 소속 없는 옵션이 생기거나 옵션만 남는 상태가 구조적으로 발생하지 않게 되었습니다. 클래스 수가 줄어 변경 지점도 함께 감소했습니다.

## 2. 지연 로딩 환경의 N+1 대응
### 문제

`Item` 목록 조회 시 각 항목의 옵션을 지연 로딩하면, 조회된 Item 수만큼 옵션 조회 쿼리가 추가로 발생합니다(N+1).

### 판단

즉시 로딩은 옵션이 필요 없는 조회에서도 항상 함께 조회되어 부적합했고, fetch join은 컬렉션 조인 시 페이징 처리에 제약이 있었습니다. 지연 로딩을 유지하면서 IN 절로 묶어 조회하는 `@BatchSize`를 선택했습니다.

### 적용
```java
@BatchSize(size = 100)
@OneToMany(mappedBy = "item", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Option> options = new ArrayList<>();
```
### 결과

옵션 조회 쿼리가 `Item` 개수에 비례하지 않고, 배치 크기 단위로 묶여 실행됩니다.
