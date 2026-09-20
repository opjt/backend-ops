---
title: "InnoDB 버퍼 풀이 있는데도 Redis 캐시를 쓰는 이유는?"
preview: "InnoDB는 자주 읽은 데이터 페이지를 이미 메모리에 캐시합니다. 그런데도 많은 서비스가 Redis를 따로 둡니다. 버퍼 풀만으로는 왜 부족할까요?"
tags: [mysql, redis, cache, database]
---

둘은 **캐시하는 대상이 다릅니다.** InnoDB 버퍼 풀은 디스크에서 읽은 **데이터·인덱스 페이지**(기본 16KB)를 메모리에 올려 디스크 I/O를 줄입니다. Redis는 애플리케이션이 **가공을 끝낸 결과값**을 키-값으로 저장해 쿼리 실행 자체를 건너뜁니다.

**버퍼 풀이 줄여주지 못하는 비용**

페이지가 메모리에 있어도 아래 비용은 그대로 남습니다.

- SQL 파싱과 실행 계획 수립
- B+Tree 인덱스 탐색, row 조립, MVCC 처리
- 조인·정렬·집계 같은 계산
- 애플리케이션과 DB 사이의 커넥션과 네트워크 왕복

MySQL 8.0부터는 쿼리 캐시가 제거되어, DB가 쿼리 결과를 대신 캐시해주지 않습니다.

**Redis를 썼을 때 얻는 이득**

- **계산과 DB 요청을 건너뜁니다.** 무거운 조인·집계 결과를 저장해두면 같은 요청은 키 조회 한 번으로 끝나고, 읽기 요청이 DB의 커넥션과 CPU까지 도달하지 않습니다.
- **DB 서버 밖에서 확장할 수 있습니다.** 버퍼 풀은 DB 서버 한 대의 메모리에 묶이지만, Redis는 노드를 늘리거나 클러스터로 구성할 수 있습니다.

읽기 복제본으로도 읽기 부하를 나눌 수 있지만, 복제본은 쿼리를 그대로 다시 실행하고 복제 지연도 생깁니다. 같은 결과를 반복해서 계산하는 부하를 줄이는 데는 결과를 재사용하는 Redis가 더 직접적입니다.

가장 흔한 사용 방식은 TTL로 오래된 값을 정리하는 **cache-aside**입니다. (`encode`, `decode`는 직렬화 함수로 생략했습니다.)

```go
func GetUser(ctx context.Context, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)

    // 캐시 조회. redis.Nil은 단순한 미스
    v, err := rdb.Get(ctx, key).Result()
    if err == nil {
        return decode(v), nil
    }
    if !errors.Is(err, redis.Nil) {
        // 장애면 기록하고 DB로 폴백
        log.Printf("redis: %v", err)
    }

    // 미스: DB 조회 후 TTL과 함께 저장
    u, err := db.QueryUser(ctx, id)
    if err != nil {
        return nil, err
    }
    rdb.Set(ctx, key, encode(u), 10*time.Minute)
    return u, nil
}
```

**주의할 점**

Redis는 공짜가 아닙니다. 원본과 캐시가 어긋나는 **일관성 문제**, 캐시가 한꺼번에 만료될 때 DB로 요청이 몰리는 **스탬피드**, 운영할 시스템이 하나 늘어나는 비용이 따라옵니다. Redis가 죽으면 모든 요청이 DB로 폴백되므로 그 부하를 감당할 수 있는지도 봐야 합니다.

그래서 먼저 버퍼 풀이 충분한지 확인합니다. `Innodb_buffer_pool_reads`(디스크에서 읽은 횟수)를 `Innodb_buffer_pool_read_requests`(전체 논리적 읽기 횟수)로 나눈 값을 1에서 빼면 히트율입니다. 히트율이 높은데도 DB가 느리다면 병목은 I/O가 아니라 쿼리 실행이나 커넥션일 가능성이 큽니다.

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
```

인덱스와 쿼리를 먼저 튜닝하고, 그래도 같은 결과를 반복해서 계산하는 부하가 남을 때 Redis를 도입하는 순서가 안전합니다.

## 참고

- [MySQL 공식 문서 — InnoDB Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)
- [MySQL 공식 문서 — What Is New in MySQL 8.0 (쿼리 캐시 제거)](https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html)
- [AWS 백서 — Database Caching Strategies Using Redis](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/welcome.html)
