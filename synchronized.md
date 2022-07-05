## 동시성
- 상품 리뷰에서 하나의 상품에는 유저 당 하나의 리뷰만 작성하도록 구현이 필요했다.
```java
@Transactional
public class ReviewServiceImpl implements ReviewService {
    ...
    @Override
    public Long saveReview(MultipartFile multipartFile, Review review, Long productId) {
        User user = getValidUser();
        Product product = getValidProduct(productId);

        if (!isOrderCompleteUser(user.getId(), product.getId()))
            throw new OrderNotCompletedException();

        return updateAndSaveReview(review, multipartFile, user, product).getId();
    }
    
    private Review updateAndSaveReview(Review review, MultipartFile multipartFile, User user, Product product) {
        String imageUrl = saveFileAndGetUrl(multipartFile);

        review.updateUser(user);
        review.updateProduct(product);
        review.updateReviewImageUrl(imageUrl);

        checkReviewDuplication(user, product.getId());
        return reviewRepository.save(review);
    }
    
    private void checkReviewDuplication(User user, Long productId) {
        reviewRepository.findByUserAndProductId(user, productId)
                .ifPresent(r -> {
                    throw new ReviewDuplicateException();
                });
    }
    ...
}
```
- 초기에 위와 같이 코드를 작성했는데, 만약 같은 user와 product의 여러 스레드가 checkReviewDuplication를 동시에 접근하여 예외처리를 피하게 되면 어떻게 될 지 궁금해서 테스트를  진행해보았다.
```java
public class ReviewServiceImplIntegrityTest extends IntegrationBase {
    ...
    @Test
    @DisplayName("리뷰 동시 저장 시 중복 저장되지 않음")
    void reviewSaveConcurrencyTest() throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(5);
        final int numberOfThreads = 5;
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        for (int j = 0; j < 5; j++) {
            service.execute(() -> {
                try {
                    setSecurityName("test");
                    Review review = Review.builder().rating(5).content("good").build();
                    when(orderRepository.findByUserIdAndProductIdAndStatus(anyLong(), anyLong(), eq(Status.COMPLETED))).thenReturn(List.of(mock(Orders.class)));
                    //테스트 메소드
                    reviewService.saveReview(null, review, product.getId());

                } catch (Exception e) {
                    e.printStackTrace();
                }
                SecurityContextHolder.clearContext();
                latch.countDown();
            });
        }
        latch.await();

        ReviewSearchRequest reviewSearchRequest = new ReviewSearchRequest(null, product.getId(), null, null);
        Page<Review> reviews = reviewService.findReviews(reviewSearchRequest, PageRequest.of(0, 100000));
        long count = reviews.getTotalElements();

        System.out.println("리뷰 수 : " + count);
        assertThat(count).isEqualTo(1L);
    }
}
```
![image](https://user-images.githubusercontent.com/63232876/177269862-86106974-ce7d-4fc4-bbee-4492bec389ac.png)
- 결과는 예외처리가 되지 않고 동시에 저장되었다.
- 이를 해결하는 두 가지 방안을 찾을 수 있었다.
    1. 기존의 비식별관계를 통해 만들어진 reviewId 대신 userId와 productId를 복합키로 사용하는 것이다. DB 수준에서 동기화가 이루어지기 때문에 중복확인을 넘어가더라도 키가 중복되기 때문에 동시 저장을 막을 수 있다. 하지만 기존의 테이블과 함께 변경해야할 코드가 많아 다음 방식을 사용하였다.
    2. saveReview() 메소드를 동기화하는 것이다. 공유 메소드의 lock을 걸기 때문에 다른 스레드의 접근이 불가하여 동시성 문제를 막을 수 있다.
- 그래서 synchronized를 사용했으나 그래도 문제는 발생하였다. 원인은 @Transactional과 synchronized의 순서에 있었다.
- @Transactional은 AOP이므로 proxy를 사용한다. 만약 transaction 내부에서 synchronized를 사용한다면 transaction의 시작점과 commit이 되는 끝점에서는 동기화가 이루어지지 않는다.
    - 만약 스레드가 메소드 A를 실행한다고 하면
    - Thread1 : S(시작) - A - C(커밋)
    - Thread2 : S(시작) ----- A - C(커밋)
- 이런 식으로 커밋과 메소드 실행 사이의 동기화가 이루어지지 않아 원하는 결과가 나오지 않을 수 있다.
- 이를 해결하기 위해 동기화 메소드 내부에서 @Transactional을 사용하여 문제를 해결하였다.
```java
public class ReviewServiceImpl implements ReviewService {
    ...
    @Override
    public synchronized Long saveReview(MultipartFile multipartFile, ReviewSaveRequest request) {
        User user = getValidUser();
        String imageUrl = saveFileAndGetUrl(multipartFile);
        Product product = getValidProduct(request.getProductId());
        Review review = request.toEntity(imageUrl, user, product);
        return synchSaveReview(review).getId();
    }
    
    @Transactional
    public Review updateAndSaveReview(Review review, MultipartFile multipartFile, User user, Product product) {
        ...

        checkReviewDuplication(user, product.getId());
        return reviewRepository.save(review);
    }
    
    private void checkReviewDuplication(User user, Long productId) {
        ...
    }
    ...
}
```
![image](https://user-images.githubusercontent.com/63232876/177271223-21d436ab-47c4-496f-af22-4506bf09e44c.png)
- 이후 진행한 테스트에서 통과하였다.
