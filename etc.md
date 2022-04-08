# 기타
## CORS(Cross-Origin Resource Sharing)
- React에서 Spring으로 axios 요청을 했을 때 CORS 관련 오류를 처음 마주했다.
### CORS란?
- 브라우저에서 다른 출처의 리소스를 공유하는 방법이다.
- CORS 이슈는 브라우저가 해당 사이트를 신뢰하지 못해서 발생한다.
- 만약 A 사이트의 로그인 정보를 쿠키로 저장한 후 B 사이트로 이동한다면 B 사이트에서 A 사이트에서 사용한 정보를 탈취할 위험이 있다. 이를 막기 위해 CORS 관련 이슈가 발생하는 것이다. 정확하게는 SOP가 요청을 막는 것이다.
### SOP(Same-origin policy)란?
- Origin이 같은 페이지끼리만 자원을 공유하도록 하는 정책이다.
- Origin은 URL Schema, Hostname, Port로 구성되어 있다.

### 해결 방법
- 이 프로젝트에서는 백엔드에서 요청을 허락할 다른 출처를 표시하여 해결하였다.
```JAVA
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    ..
    ..
    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(Arrays.asList("http://localhost:3000"));
        configuration.setAllowedMethods(Arrays.asList("HEAD", "GET", "POST", "PUT", "DELETE", "PATCH"));
        configuration.setAllowedHeaders(Arrays.asList("Authorization", "Cache-Control", "Content-Type"));
        configuration.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

## 개발 환경
### 서버 실행
- 프론트엔드에서 백엔드의 코드를 로컬에서 실행하여 테스트하고 있다.
- 백엔드에서 변경된 사항이 프론트엔드로 바로 반영되지 않고 매번 데이터를 재삽입해야하는 문제가 있다.
- 나를 포함해 서로 협업을 한 적이 없어서 가장 무식한(?) 방법으로 진행하고 있는데, AWS를 공부해서 AWS로 먼저 배포를 하고 개발을 진행하면 좋을 것 같다.
- ERD 만들고 git 설정하고 서버 배포하고.. 본격적인 개발을 시작하기 전에 할 일이 은근히 많다.


## DTO
### Service 반환
```java
public class CategoryResponse{
    ..
    ..
    public static CategoryResponse createCategoryResponse(List<Category> categories) {
        Category rootCategory = categories.get(0);
        Map<Long, CategoryResponse> map = new HashMap<>();
        categories.stream().forEach(c -> {
            CategoryResponse response = new CategoryResponse(c);
            map.put(c.getId(), response);
            if (c.getId() != rootCategory.getId())
                map.get(c.getParent().getId()).getSubCategories().add(response);
        });
        return map.get(rootCategory.getId());
    }
    ..
}
```
- 상위 카테고리 DTO에 하위 카테고리 DTO를 리스트로 추가하는 코드이다.
- 처음에는 service에 해당 코드를 구현했다. 그런데 CategoryResponse로 반환을 하면 레이어 간의 의존성이 생기는 것 같아서 service에서 하위 카테고리가 설정되지 않은 정렬된 List\<Category>로 controller에 넘긴 후 DTO class에서 변환하는 것으로 바꾸었다.


### DTO 변환
- 프로젝트를 하면서 DTO를 어디서 변환을 해야하는지 고민이 있었다. 두가지 방법이 생각했었다.
1. controller에서 entity로 변환
    - 이 방법은 레이어 간의 의존성이 줄어든다는 장점이 있다.
    - 하지만 entity를 controller에서 사용하면 service 호출 시 값이 변경될 위험성이 존재한다.
2. controller에서 변환 없이 DTO로 전달
    - 이 방법은 entity가 노출되지 않아 더 안전하게 값을 전달할 수 있지만, 레이어간의 의존성이 증가한다.
- 두가지 모두 장단점이 존재한다. 프로젝트에는 두 번째 방법을 사용하고 있다. 이 프로젝트에서는 단순한 CRUD가 많다. 하나의 컨트롤러 메소드가 하나의 서비스 메소드를 사용하는 경우가 많을 것 같아 두 번째 방법을 사용하였다.
- 그러다 https://github.com/woowacourse-teams 우테코에서 다른 분들이 진행한 프로젝트를 참고하다가 service에서 DTO 패키지를 하나 더 사용하는 것을 보았다. 이 방법을 사용하면 코드의 양이 증가하기는 하지만 위의 단점을 해소할 수 있는 있기에 좋은 방법이라고 생각한다. 시간이 된다면 바꿔보거나 다음 프로젝트에서 이 방식을 사용해보고 싶다.