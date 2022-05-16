# Fail-safe Default
- Fail-safe Default는 서비스의 디폴트는 접근 불가라는 보안 디자인 원칙 중 하나이다.
- 컴퓨터 보안 강의 시간에 배운 내용이라 프로젝트에 적용해 보고 싶어 리팩토링을 해보았다.
```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
  ...
  @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                ...
                
                .authorizeRequests() // 요청에 대한 사용권한 체크
                .antMatchers(HttpMethod.POST, "/categories")
                .hasAnyRole("ADMIN")
                .antMatchers(HttpMethod.DELETE, "/categories/*")
                .hasAnyRole("ADMIN")
                .antMatchers(HttpMethod.PATCH, "/categories")
                .hasAnyRole("ADMIN")
                .antMatchers(HttpMethod.POST, "/follows", "/sellers", "/products/**", "/stories/**", "/options/**")
                .hasAnyRole("USER", "ADMIN") // user, admin post 요청만 허용
                .antMatchers(HttpMethod.DELETE, "/follows", "/products/**", "/stories/**", "/options/**")
                .hasAnyRole("USER", "ADMIN") // user, admin delete 요청만 허용
                .antMatchers(HttpMethod.PUT, "/products", "/options/**")
                .hasAnyRole("USER", "ADMIN")
                .antMatchers(HttpMethod.PATCH, "/stories/**")
                .hasAnyRole("USER", "ADMIN")
                .antMatchers("/users/password/new", "/users/signup", "/oauth2/**")
                .hasAnyRole("GUEST") // guest 요청만 허용
                .antMatchers()
                .authenticated()// 인증된 요청만 허용
                .anyRequest().permitAll() // 그 외 모든 요청 허용
                
                ...
    }
  ...
}
```

- 현재 위와 같이 모든 요청을 허용하고 필요할 때만 요청을 거부하는 방식을 사용하고 있다.
- Fail-safe default와는 정반대로 구현을 하고 있었다.

```java
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
  ...
  @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                ...
                
                .authorizeRequests() // 요청에 대한 사용권한 체크
                ...
                .antMatchers("/users/password/new", "/users/signup", "/oauth2/**")
                .hasAnyRole("GUEST") // guest 요청만 허용
                .antMatchers()
                .authenticated()// 인증된 요청만 허용

                .antMatchers(HttpMethod.GET, "/categories/**", "/options/**", "/products/**", "/stories/**",
                        "/follows/**", "/sellers/**", "/users/**", "/orders/**")
                .permitAll()
                .antMatchers(HttpMethod.POST, "/users/login")
                .anonymous()
                .antMatchers("/emails/**")
                .permitAll()// 모든 요청 허용

                .anyRequest().denyAll()//그외 모든 요청 거부
                
                ...
    }
  ...
}
```

- 다음과 같이 코드를 기본적으로 접근을 거부한 후 필요한 API의 설정을 추가하는 것으로 코드를 변경하였다.

