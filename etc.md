# 기타
## 개발 환경
### 서버 실행
- 프론트엔드에서 백엔드의 코드를 로컬에서 실행하여 테스트하고 있다.
- 백엔드에서 변경된 사항이 프론트엔드로 바로 반영되지 않고 매번 데이터를 재삽입해야하는 문제가 있다.
- 나를 포함해 서로 협업을 한 적이 없어서 가장 무식한(?) 방법으로 진행하고 있는데, AWS를 공부해서 AWS로 먼저 배포를 하고 개발을 진행하면 좋을 것 같다.
- ERD 만들고 git 설정하고 서버 배포하고.. 본격적인 개발을 시작하기 전에 할 일이 은근히 많다.


## 레이어 의존성
### DTO 변환
```java
    private CategoryResponse createCategoryResponse(List<Category> categories) {
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
```
- 상위 카테고리 DTO에 하위 카테고리 DTO를 리스트로 추가하는 코드이다.
- 처음에는 service에 해당 코드를 구현했다. 그런데 CategoryResponse로 반환을 하면 레이어 간의 의존성이 생기는 것 같아서 service에서 하위 카테고리가 설정되지 않은 정렬된 List\<Category>로 넘긴 후 controller에서 변환하는 것으로 바꿨다.
- controller에서 리스트를 받을 때 정렬이 되었다고 가정하고 받는 거라 의존성이 완전히 해소되지는 않은 것 같다. 그리고 이런 로직을 controller에 구현을 하는 것이 맞는지는 잘 모르겠다...