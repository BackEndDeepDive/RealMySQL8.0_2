<div align="center">

# Deep Dive  
Deep Dive into Real MySQL 8.0(2)

--- 
</div>

## Contents
- [들어가며](#들어가며)
- [파티셔닝이란](#파티셔닝이란)
- [RANGE Partitioning](#range-partitioning)
- [LIST Partitioning](#list-partitioning)
- [HASH Partitioning](#hash-partitioning)
- [KEY Partitioning](#key-partitioning)
- [어떤 파티셔닝을 선택해야 할까](#어떤-파티셔닝을-선택해야-할까)
- [마치며](#마치며)

---

## 들어가며

서비스가 성장하면 테이블의 데이터가 수천만 건, 수억 건으로 늘어나면 단순한 SELECT 쿼리조차 느려지기 시작한다. 인덱스를 아무리 잘 설계해도 테이블 자체가 너무 크면 한계가 있다.

이때 고려할 수 있는 방법 중 하나가 파티셔닝이다. MySQL은 RANGE, LIST, HASH, KEY 네 가지 파티셔닝 방식을 제공하는데, 각각 언제 쓰는지, 어떤 장단점이 있는지 MySQL 공식 문서를 기반으로 정리해보았다.

---

## 파티셔닝이란

파티셔닝은 하나의 테이블을 여러 개의 물리적인 조각으로 나누어 저장하는 기법이다. MySQL 서버 입장에서는 데이터를 별도의 파티션으로 분리해서 저장하지만, 어플리케이션 입장에서는 여전히 하나의 테이블처럼 읽고 쓸 수 있다.

> Partition Pruning 란?
> 쿼리 실행 시 조건에 맞는 파티션만 스캔하고 나머지는 건너뛰는 최적화 기법이다. 파티셔닝의 성능 이점 대부분은 이 Partition Pruning에서 나온다.

파티셔닝을 사용하면 다음과 같은 이점을 얻을 수 있다.

1. 쿼리 성능이 향상된다. 전체 테이블이 아닌 특정 파티션만 스캔하므로 범위 검색이 빨라진다.
2. 데이터 관리가 쉬워진다. 오래된 데이터를 담고 있는 파티션만 DROP하면 대량 삭제를 빠르게 처리할 수 있다.
3. 가용성이 높아진다. 특정 파티션에 문제가 생겨도 다른 파티션의 데이터는 영향을 받지 않는다.

---

## RANGE Partitioning

RANGE 파티셔닝은 컬럼 값이 지정한 범위 내에 속하는지에 따라 파티션을 결정하는 방식이다. 범위는 연속적이어야 하고 서로 겹치면 안 된다.

> VALUES LESS THAN 란?
> RANGE 파티션의 범위를 정의하는 연산자다. 지정한 값 미만의 데이터가 해당 파티션에 저장된다.

### 언제 사용할까?

날짜나 시간 기반의 데이터를 다룰 때 가장 적합하다. 로그 테이블, 주문 테이블, 이벤트 테이블처럼 시계열 데이터가 대표적이다. 오래된 데이터를 주기적으로 삭제해야 하는 경우에도 유용하다. 파티션 단위로 DROP하면 DELETE보다 훨씬 빠르다.

### 사용 예시

```sql
CREATE TABLE orders (
    id INT NOT NULL,
    order_date DATE NOT NULL,
    customer_id INT NOT NULL,
    amount DECIMAL(10,2)
)
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

2022년 주문은 p2022 파티션에, 2023년 주문은 p2023 파티션에 저장된다. MAXVALUE는 정의된 범위를 벗어나는 모든 값을 받아주는 역할을 한다.
- 장점: 오래된 파티션을 DROP PARTITION으로 빠르게 삭제할 수 있다. 범위 조건 쿼리에서 Partition Pruning이 효과적으로 동작한다. 월별, 연도별 등 직관적인 기준으로 관리할 수 있다.
- 단점: 새 파티션은 기존 파티션 목록의 맨 끝에만 추가할 수 있다. 중간에 파티션을 추가하려면 REORGANIZE PARTITION을 사용해야 한다. MAXVALUE 파티션이 없으면 범위를 벗어나는 값이 들어올 때 에러가 발생한다.

---

## LIST Partitioning

LIST 파티셔닝은 RANGE와 비슷하지만, 연속적인 범위가 아닌 이산적인 값들의 목록을 기준으로 파티션을 나눈다.

### 언제 사용할까?

지역, 카테고리, 상태값처럼 명확하게 구분되는 값이 있을 때 적합하다. 특정 그룹의 데이터만 따로 관리하거나 조회해야 하는 경우에 유용하다.

### 사용 예시

```sql
CREATE TABLE stores (
    id INT NOT NULL,
    name VARCHAR(100),
    region_code VARCHAR(10) NOT NULL
)
PARTITION BY LIST COLUMNS (region_code) (
    PARTITION p_asia VALUES IN ('KR', 'JP', 'CN', 'TW'),
    PARTITION p_europe VALUES IN ('UK', 'DE', 'FR', 'IT'),
    PARTITION p_america VALUES IN ('US', 'CA', 'MX', 'BR')
);
```

한국, 일본, 중국, 대만 매장은 p_asia 파티션에, 미국, 캐나다, 멕시코, 브라질 매장은 p_america 파티션에 저장된다.

- 장점: 논리적으로 관련된 데이터를 하나의 파티션으로 그룹핑할 수 있다. 특정 지역이나 카테고리의 데이터만 빠르게 조회하거나 삭제할 수 있다.
- 단점: LIST 파티션을 삭제하면 해당 파티션에 정의되었던 값들은 더 이상 테이블에 삽입할 수 없다. 목록에 정의되지 않은 값이 들어오면 에러가 발생한다. 값 목록을 직접 관리해야 하는 번거로움이 있다.

---

## HASH Partitioning

HASH 파티셔닝은 사용자가 정의한 표현식의 결과값을 해싱하여 파티션을 결정하는 방식이다. 데이터를 균등하게 분산시키는 것이 주 목적이다.

> MOD 연산방식
> HASH 파티셔닝에서 파티션 번호를 결정하는 방식이다. 표현식 결과를 파티션 개수로 나눈 나머지가 파티션 번호가 된다. N = MOD(expr, num)

### 언제 사용할까?

데이터를 여러 파티션에 균등하게 분산시키고 싶을 때 사용한다. 특정 파티션에 데이터가 몰리는 핫스팟 현상을 방지하고 싶을 때 유용하다. 명확한 범위나 목록으로 구분하기 어려운 경우에 적합하다.

### 사용 예시

```sql
CREATE TABLE sessions (
    id INT NOT NULL,
    user_id INT NOT NULL,
    created_at DATETIME NOT NULL
)
PARTITION BY HASH(user_id)
PARTITIONS 8;
```

user_id를 8로 나눈 나머지에 따라 파티션이 결정된다. user_id가 1이면 파티션 1, user_id가 10이면 파티션 2에 저장된다.

- 장점: 데이터가 파티션에 균등하게 분산된다. 설정이 간단하다. 파티션 개수만 지정하면 된다.
- 단점: RANGE나 LIST처럼 특정 파티션만 DROP할 수 없다. 파티션을 줄이려면 COALESCE PARTITION을 사용해야 한다. 범위 조건 쿼리에서 Partition Pruning 효과가 없다. 모든 파티션을 스캔해야 할 수 있다.

---

## KEY Partitioning

KEY 파티셔닝은 HASH 파티셔닝과 비슷하지만, 해싱 함수를 사용자가 정의하는 대신 MySQL이 내부적으로 제공한다.

### 언제 사용할까?

Primary Key나 Unique Key를 기준으로 데이터를 분산시키고 싶을 때 적합하다. 정수가 아닌 컬럼으로도 균등 분산이 필요할 때 유용하다. MySQL이 제공하는 해싱 함수가 컬럼 타입에 관계없이 정수 결과를 보장하기 때문이다.

### 사용 예시

```sql
CREATE TABLE members (
    id INT NOT NULL PRIMARY KEY,
    email VARCHAR(100) NOT NULL,
    joined DATE NOT NULL
)
PARTITION BY KEY(joined)
PARTITIONS 6;
```

KEY 파티셔닝에서는 DATE, TIME, DATETIME 컬럼을 별도의 변환 없이 바로 파티셔닝 키로 사용할 수 있다. HASH 파티셔닝에서는 YEAR()나 TO_DAYS() 같은 함수로 정수 변환이 필요하다.

- 장점: 정수가 아닌 타입도 직접 파티셔닝 키로 사용할 수 있다. MySQL이 해싱을 자동으로 처리하므로 설정이 간단하다.
- 단점: HASH와 마찬가지로 개별 파티션을 DROP할 수 없다. 범위 조건 쿼리에서 Partition Pruning 효과가 없다.

---

## 어떤 파티셔닝을 선택해야 할까

네 가지 파티셔닝 방식을 비교하면 다음과 같다.

RANGE는 연속적인 범위를 기준으로 나누고, LIST는 이산적인 값 목록을 기준으로 나눈다. 이 둘은 Partition Pruning이 효과적으로 동작하고 파티션 단위 DROP이 가능하다.

HASH와 KEY는 데이터를 균등하게 분산시키는 것이 목적이다. HASH는 사용자가 정의한 표현식을, KEY는 MySQL 내장 해시 함수를 사용한다는 차이가 있다. 이 둘은 파티션 단위 DROP이 불가능하고 범위 쿼리에서 Partition Pruning 효과가 제한적이다.

실무에서 선택 기준을 정리하면 이렇다.

1. 날짜나 시간 기반 조회가 많다면 RANGE를 선택한다. 로그 테이블이나 주문 테이블처럼 시계열 데이터에 적합하다.
2. 지역, 카테고리 등 명확한 구분값이 있다면 LIST를 선택한다. 특정 그룹의 데이터만 관리해야 할 때 유용하다.
3. 데이터를 균등하게 분산시키는 것이 목표라면 HASH나 KEY를 선택한다. 핫스팟 방지가 필요하거나 범위 조건보다 등가 조건 쿼리가 많은 경우에 적합하다.

---

## 마치며

파티셔닝은 대용량 테이블을 관리하는 효과적인 방법이지만, 모든 상황에 적합한 것은 아니다. 파티셔닝 키가 쿼리 조건에 포함되지 않으면 오히려 모든 파티션을 스캔해야 해서 성능이 나빠질 수 있다.

또한 파티셔닝은 하나의 DB 서버 안에서 테이블을 나누는 것이다. 서버 자체의 한계를 넘어서려면 여러 DB 서버에 데이터를 분산하는 샤딩을 고려해야 한다.

## Ref

- [MySQL 8.0 Reference Manual - Partitioning Types](https://dev.mysql.com/doc/refman/8.0/en/partitioning-types.html)  
- [MySQL 8.0 Reference Manual - RANGE Partitioning](https://dev.mysql.com/doc/refman/8.0/en/partitioning-range.html)  
- [MySQL 8.0 Reference Manual - HASH Partitioning](https://dev.mysql.com/doc/refman/8.0/en/partitioning-hash.html)  
- [MySQL 8.0 Reference Manual - Restrictions and Limitations on Partitioning](https://dev.mysql.com/doc/refman/8.0/en/partitioning-limitations.html)  

