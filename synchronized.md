## 동시성
- 상품 리뷰에서 하나의 상품에는 유저 당 하나의 리뷰만 작성하도록 구현이 필요했다.
```java
public class ReviewServiceImpl implements ReviewService {
    ...
    @Override
    public Long saveReview(MultipartFile multipartFile, ReviewSaveRequest request) {
        User user = getValidUser();
        String imageUrl = saveFileAndGetUrl(multipartFile);
        Product product = getValidProduct(request.getProductId());
        checkReviewDuplication(user.getId(), product.getId());
        Review save = reviewRepository.save(request.toEntity(imageUrl, user, product));
        return save.getId();
    }
    private void checkReviewDuplication(Long userId, Long productId){
        reviewRepository.findByUserIdAndProductId(userId, productId)
                .ifPresent(r->{
                    throw new ReviewDuplicateException();
                });
    }
    ...
```
- 초기에 위와 같이 코드를 작성했는데, 만약 같은 userId, productId의 여러 스레드가 checkReviewDuplication를 동시에 접근하여 예외처리를 피하게 되면 어떻게 될 지 궁금해서 테스트를  진행해보았다.
```java
public class ReviewServiceImplIntegrityTest extends IntegrationBase {
    ...
    @Test
    @DisplayName("리뷰 동시 저장 시 중복 저장 확인")
    void reviewSaveConcurrencyTest() throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(10);

        for (int j = 0; j < 5; j++) {
            int finalJ = j;
            int numberOfThreads = 2;
            CountDownLatch latch = new CountDownLatch(numberOfThreads);
            for (int i = 0; i < 2; i++) {
                service.execute(() -> {
                    try {
                        setSecurityName("test");
                        ReviewSaveRequest review = new ReviewSaveRequest(5, ids.get(finalJ), "good"); // 리뷰
                        //테스트 메소드
                        reviewService.saveReview(null, review);
                        SecurityContextHolder.clearContext();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    latch.countDown();
                });
            }
            latch.await();
            int count = reviewRepository.countReviewsByProductId(ids.get(finalJ));
            System.out.println("리뷰 수 : " + count); // 리뷰 동시 저장 시 2
            assertThat(count).isEqualTo(2);
        }
    }
```
![image](https://user-images.githubusercontent.com/63232876/167418388-c0ca7aed-8d5d-432b-8bec-aa56e76dfc97.png)
- 결과는 예외처리가 되지 않고 동시에 저장되었다.
- 이를 해결하는 두 가지 방안을 찾을 수 있었다.
- 첫 번째는 기존의 비식별관계를 통해 만들어진 reviewId 대신 userId와 productId를 복합키로 사용하는 것이다. DB 수준에서 동기화가 이루어지기 때문에 중복확인을 넘어가더라도 키가 중복되기 때문에 동시 저장을 막을 수 있다. 하지만 기존의 테이블과 함께 변경해야할 코드가 많아 다음 방식을 사용하였다.
- 두 번째 saveReview() 메소드를 동기화하는 것이다. 공유 메소드의 lock을 걸기 때문에 다른 스레드의 접근이 불가하여 동시성 문제를 막을 수 있다.
```java
public class ReviewServiceImpl implements ReviewService {
    ...
    @Override
    public Long saveReview(MultipartFile multipartFile, ReviewSaveRequest request) {
        User user = getValidUser();
        String imageUrl = saveFileAndGetUrl(multipartFile);
        Product product = getValidProduct(request.getProductId());
        Review review = request.toEntity(imageUrl, user, product);
        return synchSaveReview(review).getId();
    }
    
    private synchronized Review synchSaveReview(Review reviewWithUserAndProduct) {
        checkReviewDuplication(reviewWithUserAndProduct.getUser().getId(), reviewWithUserAndProduct.getProduct().getId());
        return reviewRepository.save(reviewWithUserAndProduct);
    }
    ...
}
```
![image](https://user-images.githubusercontent.com/63232876/167426080-fca375bb-27f9-40cb-8956-2c013203de77.png)
- 이후 assertThat(count).isEqualTo(1)로 변경한 테스트에서 모두 통과하였다.
