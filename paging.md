# Paging

- 기존의 작성했던 paging 코드이다.
```java
public class OrderRepositoryImpl extends QuerydslRepositorySupport
    ...
    @Override
    public Page<Orders> findByUserIdWithProductAndOptions(Long userId, Pageable pageable) {
        JPQLQuery<Orders> query = from(orders)
                .innerJoin(orders.product).fetchJoin()
                .innerJoin(orders.parentOption).fetchJoin()
                .leftJoin(orders.childOption).fetchJoin()
                .leftJoin(orders.selectionOption).fetchJoin()
                .where(orders.user.id.eq(userId));

        QueryResults<Orders> results = getQuerydsl().applyPagination(pageable, query).fetchResults();
        List<Orders> ordersList = results.getResults();
        long total = results.getTotal();
        return new PageImpl<>(ordersList, pageable, total);
    }
}
```
- 위의 코드를 사용할 때의 문제점은 leftJoin이 포함된 count 쿼리문을 사용한다는 것이다.
- leftJoin을 count에 불필요한 join이므로 count 쿼리문을 따로 작성하여 최적화가 가능하다.

```java
public class OrderRepositoryImpl extends QuerydslRepositorySupport
    ...
    @Override
    public Page<Orders> findAllByUserIdWithProductAndOptions(Long userId, Pageable pageable) {
        JPQLQuery<Orders> query = from(orders)
                .innerJoin(orders.product).fetchJoin()
                .innerJoin(orders.parentOption).fetchJoin()
                .leftJoin(orders.childOption).fetchJoin()
                .leftJoin(orders.selectionOption).fetchJoin()
                .where(orders.user.id.eq(userId));
        List<Orders> ordersList = getQuerydsl().applyPagination(pageable, query).fetch();

        JPQLQuery<Orders> countQuery = from(orders)
                .innerJoin(orders.product).fetchJoin()
                .innerJoin(orders.parentOption).fetchJoin()
                .where(orders.user.id.eq(userId));

        return PageableExecutionUtils.getPage(ordersList, pageable, () -> countQuery.fetchCount());
    }
}
```
- countQuery를 따로 구하여 최적화하였다.
- PageableExecutionUtils은 PageImpl와 같은 역할을 수행하지만, 마지막 페이지일 경우 인자로 전달된 함수가 실행되지 않아 쿼리를 한 번 줄일 수 있다.
- 여기도 문제는 있는데, applyPagination의 내부 구현을 보면 limit과 offset을 사용한다.
- offset을 사용하면 해당 offset까지 데이터를 모두 조회하기 때문에 속도가 느려진다.
![image](https://user-images.githubusercontent.com/63232876/169697408-0a9eb318-ffb8-4375-9322-a24a885777b9.png)
- 이를 보완한 방식이 커버링 인덱스이다.
- 모든 column을 모두 구하는 대신 pk만 우선 구한 후, pk에 해당하는 값만 조회하는 방식이다.
![image](https://user-images.githubusercontent.com/63232876/169697418-c3ac05db-545f-4826-af5d-031942e90f29.png)
- 하지만 queryDsl에서는 from 절의 서브쿼리를 사용할 수 없어서 쿼리를 두 개로 나눠 사용한다.
```java
public class OrderRepositoryImpl extends QuerydslRepositorySupport
    ...
    @Override
    public Page<Orders> findAllByUserIdWithProductAndOptions(Long userId, Pageable pageable) {
        List<Orders> ordersList = getPagingOrders(userId, pageable);

        JPQLQuery<Orders> countQuery = from(orders)
                .innerJoin(orders.product).fetchJoin()
                .innerJoin(orders.parentOption).fetchJoin()
                .where(orders.user.id.eq(userId));

        return PageableExecutionUtils.getPage(ordersList, pageable, () -> countQuery.fetchCount());
    }

    private List<Orders> getPagingOrders(Long userId, Pageable pageable) {
        JPAQuery<Long> idQuery = jpaQueryFactory.select(orders.id)
                .from(orders)
                .innerJoin(orders.product)
                .innerJoin(orders.parentOption)
                .where(orders.user.id.eq(userId));
        List<Long> ids = getQuerydsl().applyPagination(pageable, idQuery).fetch();

        if (CollectionUtils.isEmpty(ids)) {
            return new ArrayList<>();
        }

        JPQLQuery<Orders> query = from(orders)
                .from(orders)
                .innerJoin(orders.product).fetchJoin()
                .innerJoin(orders.parentOption).fetchJoin()
                .leftJoin(orders.childOption).fetchJoin()
                .leftJoin(orders.selectionOption).fetchJoin()
                .where(orders.id.in(ids));

        return getQuerydsl().applySorting(pageable.getSort(), query).fetch();
    }
}
```
- 다음과 같이 코드를 최적화하였다.
- 일반적인 paging보다는 빠르지만 코드의 양이 훨씬 길어지고 no offset 방식보다는 느리다. 또한 id 값이 너무 많아지면 성능상의 이슈가 발생할 수 있다.
- 여기서 더 최적화를 진행할 수 있다. 
1. Entity 대신 DTO로 조회 
    - 영속성 컨텍스트의 관리를 받지 않고 필요한 데이터만 조회하여 조회 속도를 높일 수 있다.
    - DTO에 의존적이고 재사용성이 떨어진다는 단점이 있다.
2. No offset 방식 사용
    - 기존의 방식은 offset까지 데이터를 읽고 limit까지의 데이터를 반환하는 방식이었다면 no offset 방식은 조회 시작 부분을 where절로 빠르게 찾을 수 있어 offset의 값이 커지더라도 속도가 느려지는 것을 막을 수 있다.
    - 일반적인 페이징 방식의 UI에서는 사용하기 힘들다.
    
- 출처 : https://jojoldu.tistory.com/529
