# 데이터베이스 정규화와 반정규화

## 정규화가 필요한 이유

### 1. 데이터 무결성 보장
- 데이터 중복을 제거하여 업데이트 이상(Update Anomaly) 방지
- 삽입 이상(Insertion Anomaly)과 삭제 이상(Deletion Anomaly) 방지
- 데이터의 일관성 유지

**예시**: 학생 수강 정보 테이블
```sql
-- 정규화 전
CREATE TABLE student_courses (
    student_id INT,
    student_name VARCHAR(100),
    student_address VARCHAR(200),  -- 학생 주소가 중복 저장됨
    course_id INT,
    course_name VARCHAR(100),
    professor_name VARCHAR(100)    -- 교수 정보가 중복 저장됨
);

-- 정규화 후
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100),
    student_address VARCHAR(200)
);

CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100),
    professor_name VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

### 2. 정규화 단계
1. 제1정규형(1NF): 원자값으로 구성
2. 제2정규형(2NF): 부분 함수적 종속 제거
3. 제3정규형(3NF): 이행적 함수적 종속 제거
4. BCNF: 결정자이면서 후보키가 아닌 것 제거
5. 제4정규형(4NF): 다치 종속 제거
6. 제5정규형(5NF): 조인 종속성 제거

## 반정규화를 고려해야 하는 상황

### 1. 조회 성능이 중요한 경우
- 데이터 조회가 매우 빈번한 경우
- 여러 테이블을 조인해야 하는 경우

**예시**: 주문 조회 시스템
```sql
-- 반정규화 전
SELECT o.order_id, o.order_date, c.customer_name, p.product_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;

-- 반정규화 후
CREATE TABLE order_summary (
    order_id INT PRIMARY KEY,
    order_date TIMESTAMP,
    customer_name VARCHAR(100),  -- 중복 허용
    product_names TEXT,          -- 제품명들을 하나의 필드에 저장
    total_amount DECIMAL(10,2)   -- 계산된 총액 저장
);
```

### 2. 자주 사용되는 통계 데이터
- 실시간 집계가 필요한 경우
- 대규모 데이터에서의 집계 연산이 부담되는 경우

```sql
-- 반정규화 예시: 주문 총액을 미리 계산하여 저장
CREATE TABLE daily_sales_summary (
    date DATE PRIMARY KEY,
    total_orders INT,
    total_amount DECIMAL(10,2),
    average_order_value DECIMAL(10,2)
);
```

### 3. 반정규화 고려 시 체크포인트
1. 데이터 일관성 vs 조회 성능
2. 저장 공간 사용량
3. 데이터 업데이트 빈도
4. 트랜잭션의 복잡도

## 근거 자료

### 학술 자료
1. E.F. Codd's Paper on Normalization:
    - "A Relational Model of Data for Large Shared Data Banks" (1970)
    - [Communications of the ACM](https://dl.acm.org/doi/10.1145/362384.362685)

2. Database System Concepts (7th Edition):
    - Chapter 7: Database Design and Normalization
    - Authors: Abraham Silberschatz, Henry F. Korth, S. Sudarshan
    - [Book Link](https://www.db-book.com/)

### 기술 문서
1. Microsoft SQL Server Documentation:
    - [Database Normalization Description](https://docs.microsoft.com/en-us/office/troubleshoot/access/database-normalization-description)
    - [Design for Performance](https://docs.microsoft.com/en-us/sql/relational-databases/performance/design-for-performance)

2. Oracle Documentation:
    - [Database Design Basics](https://docs.oracle.com/cd/B19306_01/server.102/b14231/design.htm)
    - [Schema Object Management](https://docs.oracle.com/cd/B19306_01/server.102/b14200/schema.htm)

### 실제 사례 연구
1. [Stack Overflow Database Schema](https://meta.stackexchange.com/questions/2677/database-schema-documentation-for-the-public-data-dump-and-sede):
    - 대규모 시스템에서의 정규화와 반정규화 사례
    - 성능 최적화를 위한 전략

### 성능 관련 연구
- [Impact of Database Normalization on Performance](https://arxiv.org/ftp/arxiv/papers/1508/1508.05esdeveniments-socials.pdf)
- [Trade-offs in SQL Database Design](https://www.confluent.io/blog/designing-high-performance-database-schemas/)

정규화와 반정규화는 상황에 따라 적절한 균형을 찾는 것이 중요합니다. 특히 마이크로서비스 아키텍처에서는 각 서비스의 특성에 따라 다른 정규화 수준을 적용하는 것이 일반적입니다.

실무에서는 정규화된 기본 구조를 유지하면서, 성능이 중요한 특정 부분에 대해서만 선택적으로 반정규화를 적용하는 것이 좋은 접근 방법입니다.