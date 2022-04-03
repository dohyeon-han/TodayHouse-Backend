## N+1과 재귀 쿼리문
- 계층형 DB로 Category 테이블을 만들었다. 중간 계층에서 리프노드까지의 모든 경로, 혹은 루트 노드까지의 경로를 구해야했는데, 두 가지의 해결 방법이 떠올랐다.
- 첫 번째는 lazy loading으로 경로를 따라가면서 그때마다 하위 노드, 상위 노드들을 호출하는 것인데, N+1문제가 발생해서 다른 방법을 고민했다.
- 두 번째는 재귀 쿼리문으로 연관된 category를 한 번에 불러오고 service layer나 controller layer에서 정렬을 하는 방법이었다.
- 이 방법의 문제는 JPA가 재귀를 지원하지 않는다는 것이었고 native query를 이용해여 해결할 수 있었다.
```java
    @Query(value = "with recursive rec(category_id, name, depth, parent_id) as (" +
            " select category_id, name, depth, parent_id from Category where name=:categoryName" +
            " union all " +
            " select par.category_id, par.name, par.depth, par.parent_id from rec as ch, Category as par where par.category_id = ch.parent_id" +
            " ) " +
            " select * from rec as r" +
            " order by r.depth",
            nativeQuery = true)
```