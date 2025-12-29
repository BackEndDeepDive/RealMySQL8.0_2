<div align="center">

# Deep Dive  
Deep Dive into Real MySQL 8.0(2) 

--- 
</div>

## Contents
- [들어가며](#들어가며)
- [TEXT와 BLOB이란](#text와-blob이란)
- [종류와 저장 용량](#종류와-저장-용량)
- [TEXT와 BLOB의 차이점](#text와-blob의-차이점)
- [임시 테이블과 성능 이슈](#임시-테이블과-성능-이슈)
- [사용 시 주의사항](#사용-시-주의사항)
- [VARCHAR와 TEXT 중 무엇을 선택할까](#varchar와-text-중-무엇을-선택할까)
- [마치며](#마치며)

---

## 들어가며

MySQL에서 대용량 데이터를 저장해야 할 때 TEXT나 BLOB 타입을 고려하게 된다. 게시글 본문, 상품 설명, 이미지 바이너리 등 길이가 예측하기 어려운 데이터를 다룰 때 자주 등장하는 타입이다.

그런데 막상 사용하려고 하면 궁금한 점이 많다. VARCHAR와 뭐가 다른지, 성능에는 어떤 영향을 주는지, 언제 써야 하는지 등이다. 이번 글에서는 MySQL 공식 문서와 Real MySQL 8.0을 기반으로 TEXT와 BLOB의 특성을 정리해보았다.


## TEXT와 BLOB

TEXT와 BLOB은 MySQL에서 대용량 데이터를 저장하기 위한 데이터 타입이다. 둘 다 가변 길이 타입이고, 저장할 수 있는 최대 크기에 따라 TINY, 일반, MEDIUM, LONG 네 가지 종류가 있다.

> LOB (Large Object)
> 
> 대용량 객체를 의미한다. TEXT는 CLOB(Character Large Object), BLOB은 BLOB(Binary Large Object)에 해당한다. Oracle에서는 CLOB과 BLOB으로 구분하지만, MySQL에서는 TEXT와 BLOB이라는 이름을 사용한다.

TEXT는 문자열을 저장하는 대용량 컬럼이다. 문자집합과 콜레이션을 가지며, 저장된 값은 문자열로 취급된다.

BLOB은 이진 데이터를 저장하는 대용량 컬럼이다. Binary Large Object의 약자로, 이미지나 파일 같은 바이너리 데이터를 저장할 때 사용한다. 별도의 문자집합이나 콜레이션을 가지지 않는다.


## 종류와 저장 용량

TEXT와 BLOB은 각각 네 가지 종류가 있으며, 저장할 수 있는 최대 크기만 다르다.

1. TINYTEXT와 TINYBLOB은 최대 255바이트를 저장할 수 있다.

2. TEXT와 BLOB은 최대 65,535바이트, 약 64KB를 저장할 수 있다.

3. MEDIUMTEXT와 MEDIUMBLOB은 최대 16,777,215바이트, 약 16MB를 저장할 수 있다.

4. LONGTEXT와 LONGBLOB은 최대 4,294,967,295바이트, 약 4GB를 저장할 수 있다.

> 저장 가능한 최대 크기와 실제로 클라이언트와 서버 간에 전송할 수 있는 크기는 다르다. 전송 가능한 크기는 max_allowed_packet 시스템 변수에 의해 결정된다.


## TEXT와 BLOB의 차이점

TEXT와 BLOB은 저장 구조나 동작 방식에서 거의 동일하게 작동한다. 주요 차이점은 다음과 같다.

1. 문자집합 처리가 다르다. TEXT는 문자열로 취급되어 문자집합과 콜레이션을 가진다. BLOB은 바이트 문자열로 취급되어 binary 문자집합을 가지며, 정렬이나 비교 시 바이트의 숫자 값을 기준으로 한다.

2. 정렬 방식이 다르다. TEXT는 문자집합의 콜레이션을 고려하여 정렬한다. BLOB은 저장된 바이트 값을 기준으로 정렬한다.

3. 인덱스 비교 시 공백 처리가 다르다. TEXT 컬럼에 인덱스가 있으면 비교 시 뒤따르는 공백이 무시된다. 예를 들어 'a'와 'a '는 같은 값으로 취급되어 유니크 인덱스에서 중복 오류가 발생한다. BLOB은 공백도 데이터의 일부로 취급한다.

## 임시 테이블과 성능 이슈

MySQL은 쿼리를 처리하는 과정에서 내부적으로 임시 테이블을 생성해야 할 때가 있다. ORDER BY와 GROUP BY가 다른 컬럼을 사용하거나, DISTINCT와 ORDER BY가 함께 사용되는 경우 등이다.

이때 임시 테이블은 가능하면 메모리에 생성된다. 하지만 TEXT나 BLOB 컬럼이 포함되어 있으면 상황이 달라진다.

MySQL 8.0.13 이전 버전에서는 MEMORY 스토리지 엔진이 TEXT나 BLOB 타입을 지원하지 않았다. 따라서 이런 타입이 포함된 쿼리는 임시 테이블이 무조건 디스크에 생성되었고, 이는 성능 저하로 이어졌다.

MySQL 8.0.13 버전부터는 TempTable 스토리지 엔진이 도입되어 TEXT나 BLOB 컬럼을 가진 임시 테이블도 메모리에 생성할 수 있게 개선되었다.

> TempTable 스토리지 엔진
> 
> MySQL 8.0에서 도입된 내부 임시 테이블용 스토리지 엔진이다. 기존 MEMORY 엔진과 달리 가변 길이 타입을 효율적으로 처리하고, TEXT와 BLOB 타입도 지원한다. temptable_max_ram 시스템 변수로 최대 메모리 사용량을 설정할 수 있으며, 기본값은 1GB이다.

그럼에도 TEXT나 BLOB 컬럼이 포함된 쿼리는 주의가 필요하다. SELECT * 같은 쿼리로 불필요하게 대용량 컬럼을 조회하면 성능에 영향을 줄 수 있다.

만약 TEXT나 BLOB 컬럼의 일부만 필요하다면, SUBSTRING() 함수로 필요한 부분만 조회하거나 CAST() 함수로 VARCHAR로 변환하면 메모리 임시 테이블을 유도할 수 있다.


## 사용 시 주의사항

TEXT와 BLOB을 사용할 때 알아두어야 할 몇 가지 제약사항이 있다.

1. DEFAULT 값을 지정할 수 없다. TEXT와 BLOB 컬럼은 기본값을 가질 수 없다.

2. 인덱스 생성 시 길이를 지정해야 한다. TEXT나 BLOB 컬럼에 인덱스를 생성하려면 인덱스로 사용할 문자 길이를 명시해야 한다. VARCHAR처럼 전체 컬럼을 인덱싱할 수 없다.

3. max_allowed_packet 설정에 주의해야 한다. TEXT나 BLOB 컬럼을 조작하는 SQL 문장은 매우 길어질 수 있다. max_allowed_packet 시스템 변수에 정의된 값보다 큰 SQL 문장은 서버로 전송되지 못하고 오류가 발생한다. 대용량 데이터를 다루는 쿼리가 있다면 이 설정을 충분히 늘려주는 것이 좋다.

4. 정렬 시 일부만 사용된다. TEXT나 BLOB 컬럼을 정렬할 때는 max_sort_length 바이트까지만 사용된다.


## VARCHAR와 TEXT 중 무엇을 선택할까

VARCHAR와 TEXT는 실제로 저장 방식에서 큰 차이가 없다. 둘 다 값이 커지면 Off-Page로 저장된다. 그렇다면 언제 어떤 타입을 선택해야 할까.

VARCHAR를 선택하는 경우는 다음과 같다.

저장할 데이터의 최대 길이를 예측할 수 있을 때 VARCHAR가 적합하다. 이름, 이메일, 전화번호처럼 길이가 어느 정도 제한되는 데이터다. DEFAULT 값이 필요할 때도 VARCHAR를 선택해야 한다. TEXT는 기본값을 지정할 수 없기 때문이다.

TEXT를 선택하는 경우는 다음과 같다.

저장할 데이터의 길이가 예측 불가능하게 클 때 TEXT가 적합하다. 게시글 본문, 상품 설명, 로그 메시지 등이다. 레코드의 전체 크기가 64KB 제한에 걸릴 때도 TEXT를 고려해야 한다. MySQL에서 하나의 레코드는 전체 크기가 64KB를 넘을 수 없다. VARCHAR는 최대 저장 가능 크기를 포함해 이 제한에 영향을 받지만, TEXT는 포인터만 저장되므로 이 제한을 피할 수 있다.


## 마치며

TEXT와 BLOB은 대용량 데이터를 저장할 때 유용한 타입이지만, 무분별하게 사용하면 성능 문제를 일으킬 수 있다. 특히 SELECT *로 불필요하게 대용량 컬럼을 조회하거나, 임시 테이블이 디스크에 생성되는 상황을 주의해야 한다. MySQL 8.0에서는 TempTable 스토리지 엔진 도입으로 많은 부분이 개선되었지만, 여전히 대용량 데이터를 다룰 때는 쿼리 설계에 신경을 써야 한다. 필요한 컬럼만 조회하고, 가능하면 SUBSTRING()으로 필요한 부분만 가져오는 습관을 들이면 좋다고 한다. 큰 바이너리 파일을 저장해야 한다면 데이터베이스보다는 파일 시스템에 저장하고 경로만 DB에 저장하는 방식도 고려해볼 만하다. 데이터베이스는 파일 시스템만큼 대용량 파일 처리에 최적화되어 있지 않기 때문이다.


## References

- [MySQL 8.0 Reference Manual - The BLOB and TEXT Types](https://dev.mysql.com/doc/refman/8.0/en/blob.html)  
- [MySQL 8.0 Reference Manual - Optimizing for BLOB Types](https://dev.mysql.com/doc/refman/8.0/en/optimize-blob.html)
- [당근 테크 블로그 - VARCHAR vs TEXT](https://medium.com/daangn/varchar-vs-text-230a718a22a1)  
- [MySQL Blog - Externally Stored Fields in InnoDB](https://dev.mysql.com/blog-archive/externally-stored-fields-in-innodb/)  