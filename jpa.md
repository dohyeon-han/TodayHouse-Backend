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

## JPQL fetch join
```java
    @Query("select distinct st from Scrap sc inner join fetch sc.story st where sc.story in :stories and sc.user =:user")
    List<Story> findScrapedByStoriesAndUser(@Param("stories") List<Story> stories, @Param("user") User user);
```
- Story list와 User를 이용해 인자로 받은 Story list 중 유저가 스크랩한 story만을 조회하도록 코드를 구현하였는데 이런 오류가 발생하였다.
```
Caused by: org.hibernate.QueryException: query specified join fetching, but the owner of the fetched association was not present in the select list [FromElement{explicit,not a collection join,fetch join,fetch non-lazy properties,classAlias=st,role=com.todayhouse.domain.scrap.domain.Scrap.story,tableName=story,tableAlias=story1_,origin=scrap scrap0_,columns={scrap0_.story_id,className=com.todayhouse.domain.story.domain.Story}}] [select distinct st from com.todayhouse.domain.scrap.domain.Scrap sc inner join fetch sc.story st where sc.story in :stories and sc.user =:user]
```
- 문제는 fetch에 있었다.
- fetch join은 entity 그래프를 참조하여 join한 상태로 entity를 가져오기 위해 사용하는데, 해당 쿼리는 연관관계의 주인인 Scrap이 아닌 Story만 가져오기 때문에 fetch join이 필요없다.
- 따라서 fetch를 제거해주면 정상적으로 작동한다.
