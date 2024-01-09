---
title: 페이징 쿼리 파헤쳐 보기 - 1. Row count까지 같이 조회
date: 2024-01-05 12:50:44
categories: SQL
---

# 페이징 쿼리 파헤쳐 보기 - 1. Row count까지 같이 조회

사내 관리자 페이지를 개발하면서 페이징 처리할 데이터와 페이징 Row count를 한 번에 조회할 수 있는 방법을 고민했던 내용을 정리해본다

## OVER () 사용
StackOverflow에 이 내용으로 물어보면 가장 처음 나오는 결과가 OVER () 를 사용하는 방법이다.

```sql
SELECT ...
      , total_count = COUNT(*) OVER()
-- 또는, COUNT(*) OVER() AS total_count
FROM ...
ORDER BY ...
OFFSET 120 ROWS
FETCH NEXT 10 ROWS ONLY;
```
