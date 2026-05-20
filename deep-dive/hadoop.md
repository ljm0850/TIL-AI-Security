# Hadoop 정리

> RDBMS 학습 중 접한 Hadoop에 대해 간략히 정리한 노트

---

## 1. Hadoop이란?

대용량 데이터를 여러 서버에 **분산 저장**하고 **병렬 처리**하는 오픈소스 프레임워크.  
검색에 특화된 기술이 아니라, **배치 분석·집계** 작업에 강점이 있다.

---

## 2. 핵심 구성요소

| 구성요소 | 역할 |
|---|---|
| **HDFS** | 파일을 128MB 블록으로 나눠 여러 노드에 복제 저장 (기본 복제 계수 3) |
| **MapReduce** | Map(분산 처리) → Shuffle → Reduce(집계) 패러다임의 디스크 기반 처리 엔진 |
| **YARN** | 클러스터 자원(CPU/메모리)을 스케줄링하고 여러 앱을 동시 실행 |
| **Hadoop Common** | 다른 모듈을 지원하는 공통 Java 라이브러리 |

---

## 3. 주요 생태계 도구

| 도구 | 역할 |
|---|---|
| **Hive** | HDFS 데이터를 SQL로 조회 (내부적으로 MapReduce 변환) |
| **HBase** | HDFS 위의 컬럼형 NoSQL DB, 실시간 읽기/쓰기 지원 |
| **Sqoop / Flume** | RDB·로그 등 외부 데이터를 HDFS로 수집 |
| **Oozie** | Hadoop 작업 스케줄링 및 워크플로우 자동화 |
| **Spark** | YARN 위에서 동작하는 인메모리 처리 엔진, MapReduce 대비 최대 100배 빠름 |

---

## 4. 관심이 줄어든 이유

### 4-1. Apache Spark의 등장
인메모리 처리 방식으로 MapReduce 대비 압도적으로 빠름.  
Python/Scala API로 사용성도 좋아 사실상 Hadoop의 처리 계층을 대체.

### 4-2. 클라우드 관리형 서비스의 부상
Hadoop 온프레미스의 가장 큰 고통은 인프라 운영 부담이었다.

**온프레미스 Hadoop의 문제:**
- 물리 서버 구매·설치·유지보수에 전담 인력 필요
- HDFS = 저장과 처리가 결합된 구조 → 유휴 시에도 서버 상시 운영
- 확장에 수주 소요, 고정 비용(서버·전기·냉각) 지속 발생
- 결과적으로 데이터 분석보다 인프라 운영에 더 많은 시간 소요

**클라우드가 바꾼 것:**

| 변화 | 내용 |
|---|---|
| 저장·처리 분리 | S3 같은 오브젝트 스토리지에 데이터 저장, 필요할 때만 클러스터를 켜서 처리 |
| 관리형 서비스 | AWS EMR, Google Dataproc 등이 클러스터 생성·패치·장애복구 자동화 |
| 완전 추상화 | Snowflake, BigQuery 등은 SQL만으로 페타바이트 분석 가능 |

> 클라우드 관리형 서비스는 Hadoop의 경쟁자가 아니라,  
> Hadoop이 해결하려던 문제를 훨씬 쉽고 저렴하게 해결해버린 대안.

### 4-3. 기타 요인
- **운영 복잡성**: 수십~수백 대 노드, Java 기반 설정, 버전 호환성 관리 등 학습 곡선이 가파름
- **실시간 처리 수요 증가**: MapReduce는 배치 처리 특화 → Kafka + Flink/Spark Streaming 조합이 더 적합
- **Data Lakehouse 트렌드**: Delta Lake, Apache Iceberg 등 오픈 테이블 포맷 + S3가 HDFS 역할 대체

---

## 5. 현재 대안 기술 스택

| Hadoop 구성 | 현대적 대안 |
|---|---|
| HDFS | Amazon S3 / Google GCS / Azure ADLS |
| MapReduce | Apache Spark, Apache Flink |
| Hive | Presto / Trino, BigQuery, Snowflake |
| 전체 플랫폼 | Databricks, Snowflake, Google BigQuery |

---

## 6. RDBMS와의 관계 (참고)

| | RDBMS | Hadoop |
|---|---|---|
| 데이터 구조 | 정형 데이터 (테이블) | 정형·반정형·비정형 모두 |
| 처리 방식 | 실시간 쿼리 | 배치 처리 중심 |
| 확장 방식 | 수직 확장 (Scale-up) | 수평 확장 (Scale-out) |
| 주요 용도 | 트랜잭션, 빠른 조회 | 대용량 분석, ETL |
| 현재 위치 | 여전히 핵심 기술 | 클라우드 서비스로 대체 중 |

> Hadoop은 RDBMS가 감당하기 어려운 비정형·대용량 데이터 처리 문제에서 출발했지만,  
> 클라우드 시대에는 그 역할을 더 쉬운 도구들이 대신하고 있다.
