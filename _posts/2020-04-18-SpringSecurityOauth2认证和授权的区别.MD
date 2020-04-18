---
layout:     post
title:      Sprng Security Oauth2 认证和授权的区别
subtitle:   Sprng Security Oauth2
date:       2020-03-28
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Sprng
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/springsecurity.jpg)

## 1 认证（Authentication）和授权（Authorization）的区别？

你要登机，你需要出示你的 passport 和 ticket，passport 是为了证明你是张三，这就是 authentication；而 ticket是为了证明你张三确实买了票可以上飞机，这就是 authorization。

## 2 深入剖析

### 引入 pom 依赖

```java
...
	<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-oauth2</artifactId>
    </dependency>  
    <dependency>
    <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt</artifactId>
        <version>${jwt.version}</version>
    </dependency>
...    
```

### 授权服务网关入口（可以借此入口分析源码）

```java
/**
 * 获取access_token<br>
 * 这是spring-security-oauth2底层的接口，类TokenEndpoint<br>
 *
 * @param parameters
 * @return
 * @see org.springframework.security.oauth2.provider.endpoint.TokenEndpoint
 */
@PostMapping(path = "/oauth/token")
Map<String, Object> postAccessToken(@RequestParam Map<String, String> parameters);
```



### 服务器端授权配置

增加配置类 AuthorizationServerConfig 继承 AuthorizationServerConfigurerAdapter类，增加 @EnableAuthorizationServer 标注开启授权服务器功能，增加 @Configuration 标注说明配置类。

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

	/**
	 * 认证管理器
	 * 
	 * @see SecurityConfig 的authenticationManagerBean()
	 */
	@Autowired
	private AuthenticationManager authenticationManager;
	@Autowired
	private RedisConnectionFactory redisConnectionFactory;
	@Autowired
	private DataSource dataSource;
	/**
	 * 使用jwt或者redis
	 */
	@Value("${access_token.store-jwt:false}")
	private boolean storeWithJwt;
	@Autowired
	private RedisAuthorizationCodeServices redisAuthorizationCodeServices;

	/**
	 * 令牌存储
	 * 
	 * @return
	 */
	@Bean
	public TokenStore tokenStore() {
		if (storeWithJwt) {
			return new JwtTokenStore(accessTokenConverter());
		}
		return new RedisTokenStore(redisConnectionFactory);
	}

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
		endpoints.authenticationManager(this.authenticationManager);
		endpoints.tokenStore(tokenStore());
		// 授权码模式下，code存储
//		endpoints.authorizationCodeServices(new JdbcAuthorizationCodeServices(dataSource));
		endpoints.authorizationCodeServices(redisAuthorizationCodeServices);
		if (storeWithJwt) {
			endpoints.accessTokenConverter(accessTokenConverter());
		}
	}

	@Override
	public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
		security.allowFormAuthenticationForClients();
	}

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
//		clients.inMemory().withClient("system").secret("system")
//				.authorizedGrantTypes("password", "authorization_code", "refresh_token").scopes("app")
//				.accessTokenValiditySeconds(3600);

		clients.jdbc(dataSource);
	}

	@Autowired
	public UserDetailsService userDetailsService;

	/**
	 * Jwt资源令牌转换器
	 * 
	 * @return accessTokenConverter
	 */
	@Bean
	public JwtAccessTokenConverter accessTokenConverter() {
		JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
		DefaultAccessTokenConverter defaultAccessTokenConverter = (DefaultAccessTokenConverter) jwtAccessTokenConverter
				.getAccessTokenConverter();
		DefaultUserAuthenticationConverter userAuthenticationConverter = new DefaultUserAuthenticationConverter();
		userAuthenticationConverter.setUserDetailsService(userDetailsService);

		defaultAccessTokenConverter.setUserTokenConverter(userAuthenticationConverter);

		return jwtAccessTokenConverter;
	}
    
     /**
     * Jwt资源令牌转换器
     * 如果是 JWT 中增加参数或者重写了 UserDetailsService 类
     *
     * @return accessTokenConverter
     */
    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter() {
            @Override
            public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
                String userName = authentication.getUserAuthentication().getName();
                Object principal = authentication.getUserAuthentication().getPrincipal();
                String userId = "";
                int userIdentify = 0;
                if (principal instanceof LoginAppUser) {
                    // 框架自带后台
                    LoginAppUser user = (LoginAppUser)principal;
                    userId = String.valueOf(user.getId());
                } else if (principal instanceof LoginUserV3) {
                    // 360前台用户
                    LoginUserV3 user = (LoginUserV3)principal;
                    userId = user.getUserOperationId();
                }
                LinkedHashMap details = (LinkedHashMap) authentication.getUserAuthentication().getDetails();
                String userType = (String) details.get("user_type");
                String ukey = "";
                if(details.keySet().contains("ukey")){
                    ukey = (String) details.get("ukey");
                }
                String platform_type = "";
                if(details.keySet().contains("platform_type")){
                    platform_type = (String) details.get("platform_type");
                }
                /** 自定义一些token属性 ***/
                final Map<String, Object> additionalInformation = new HashMap<>();
                additionalInformation.put("user_type", userType);
                additionalInformation.put("user_id", userId);
                additionalInformation.put("ukey", ukey);
                additionalInformation.put("platform_type", platform_type);
                ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(additionalInformation);
                OAuth2AccessToken enhancedToken = super.enhance(accessToken, authentication);
                return enhancedToken;
            }
        };
        DefaultAccessTokenConverter defaultAccessTokenConverter = (DefaultAccessTokenConverter) jwtAccessTokenConverter
                .getAccessTokenConverter();
        ZyUserAuthenticationConverter zyUserAuthenticationConverter = new ZyUserAuthenticationConverter();
        Map<String, UserDetailsService> map = new HashMap<>();
        //添加UserDetails
        map.put("appUserDetailService", appUserDetailService);
        map.put("phoneUserDetailService", phoneUserDetailService);
        map.put("storeFrontUserDetailService", storeFrontUserDetailService);
        map.put("weiXinUserDetailService", weiXinUserDetailService);
        map.put("weiXinPasswordDetailService", weiXinPasswordDetailService);
        map.put("weiXinPhoneDetailService", weiXinPhoneDetailService);
        zyUserAuthenticationConverter.setUserDetailsServiceMap(map);
        // 解决多个节点部署导致token失效的问题
        jwtAccessTokenConverter.setVerifierKey("zy");
        jwtAccessTokenConverter.setSigningKey("zy");
        defaultAccessTokenConverter.setUserTokenConverter(zyUserAuthenticationConverter);

        return jwtAccessTokenConverter;
    }    
```

如果是重写了 UserDetailsService 类，则需要增加自定义 Provider（例如：MyProvider）实现 AuthenticationProvider 类。

```java
@Component
public class MyProvider implements AuthenticationProvider {

    @Autowired
    private ZyConfig zyConfig;

    @Autowired
    private AppUserDetailServiceImpl appUserDetailService;

    @Autowired
    private UserDetailServiceImpl userDetailService;

    @Autowired
    private StoreFrontUserDetailServiceImpl storeFrontUserDetailService;

    @Autowired
    private PhoneUserDetailServiceImpl phoneUserDetailService;

    @Autowired
    private WeiXinUserDetailServiceImpl weiXinUserDetailService;

    @Autowired
    private WeiXinPasswordDetailServiceImpl weiXinPasswordDetailService;

    @Autowired
    private WeiXinPhoneDetailServiceImpl weiXinPhoneDetailService;

    @Autowired
    private LogService logService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        LinkedHashMap details = (LinkedHashMap) authentication.getDetails();
        String grantType = (String) details.get(LoginDictionary.LOGIN_INFO.USER_TYPE);
        String username = "";
        if (LoginDictionary.LOGIN_APPROACH.ZY.equals(grantType)){
            // 电票360后台登陆

            username = (String) details.get(LoginDictionary.LOGIN_INFO.USERNAME);

            // 记录日志
            logService.saveLoginLog(username, "360后台登陆");
            UserDetails userDetails = appUserDetailService.loadUserByUsername(username);
            if (!SecurityUtil.pwdEncrypt(authentication.getCredentials().toString()).equals(userDetails.getPassword())){
                throw new BadCredentialsException("用户名或密码错误");
            }
            return new UsernamePasswordAuthenticationToken(userDetails,authentication.getCredentials(),userDetails.getAuthorities());

        } else if (LoginDictionary.LOGIN_APPROACH.BACK.equals(grantType)){
            // 框架自带系统管理后台登陆
            UserDetails userDetails = userDetailService.loadUserByUsername((String) details.get(LoginDictionary.LOGIN_INFO.USERNAME));

            return new UsernamePasswordAuthenticationToken(userDetails,authentication.getCredentials(),userDetails.getAuthorities());
        } else if (LoginDictionary.LOGIN_APPROACH.STOREFRONT.equals(grantType)){
            // PC端--密码登陆（无权限可以登录，分配默认权限）

            username = (String) details.get(LoginDictionary.LOGIN_INFO.USERNAME);

            // 记录日志
            logService.saveLoginLog(username,"PC端--密码登陆（无权限可以登录，分配默认权限）");

            UserDetails userDetails = storeFrontUserDetailService.loadUserByUsername(username);
            if (!SecurityUtil.pwdEncrypt(authentication.getCredentials().toString()).equals(userDetails.getPassword())){
                throw new BadCredentialsException("用户名或密码错误");
            }
            return new UsernamePasswordAuthenticationToken(userDetails,authentication.getCredentials(),userDetails.getAuthorities());

        } else if (LoginDictionary.LOGIN_APPROACH.PHONE.equals(grantType)){
            // PC端--手机登录

            username = (String) details.get(LoginDictionary.LOGIN_INFO.USERNAME);

            logService.saveLoginLog(username, "PC端--手机登录");
            UserDetails userDetails = phoneUserDetailService.loadUserByUsername(username);

//            loginByPhoneCheckPassword(authentication, userDetails);

            return new UsernamePasswordAuthenticationToken(userDetails,authentication.getCredentials(),userDetails.getAuthorities());

        } else if (LoginDictionary.LOGIN_APPROACH.WEIXIN.equals(grantType)){
            // 微信登录（扫码）

            username = (String) details.get(LoginDictionary.LOGIN_INFO.USERNAME);

            logService.saveLoginLog(username, "微信登录（扫码或授权）");

            UserDetails userDetails = weiXinUserDetailService.loadUserByUsername(username);

            return new UsernamePasswordAuthenticationToken(userDetails,authentication.getCredentials(),userDetails.getAuthorities());

        } else if (LoginDictionary.LOGIN_APPROACH.WEIXINPASSWORD.equals(grantType)){
            // 小程序--用户名密码登录

            username = (String) details.get(LoginDictionary.LOGIN_INFO.USERNAME);

            logService.saveLoginLog(username, "小程序--用户名密码登录");

            UserDetails userDetails = weiXinPasswordDetailService.loadUserByUsername(username);
            if (!SecurityUtil.pwdEncrypt(authentication.getCredentials().toString()).equals(userDetails.getPassword())){
                throw new BadCredentialsException("用户名或密码错误");
            }
            return new UsernamePasswordAuthenticationToken(userDetails,authentication.getCredentials(),userDetails.getAuthorities());

        } else if (LoginDictionary.LOGIN_APPROACH.WEIXINPHONE.equals(grantType)){
            // 小程序--手机验证码登录
            username = (String) details.get(LoginDictionary.LOGIN_INFO.USERNAME);

            logService.saveLoginLog(username, "小程序--手机验证码登录");

            UserDetails userDetails = weiXinPhoneDetailService.loadUserByUsername(username);

            // 判断是否是测试环境
//            loginByPhoneCheckPassword(authentication, userDetails);

            return new UsernamePasswordAuthenticationToken(userDetails,authentication.getCredentials(),userDetails.getAuthorities());
        }

        return null;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }

    /**
     * 手机验证码登陆--校验验证码
     *
     */
    private void loginByPhoneCheckPassword(Authentication authentication, UserDetails userDetails) {
        // 判断是否是测试环境
        String devCode = zyConfig.getCode();
        if (devCode != null && devCode.length() > 0) {
            // 测试环境，先判断输入的验证码是否等于测试环境通用验证码
            // 如果不等于则判断输入的验证码是否等于发送的验证码
            if (!authentication.getCredentials().toString().equals(MD5Utils.getMD5(devCode))){
                if (!authentication.getCredentials().toString().equals(userDetails.getPassword())) {
                    throw new BadCredentialsException("验证码错误");
                }
            }
        } else {
            // 不是测试环境
            if (!authentication.getCredentials().toString().equals(userDetails.getPassword())) {
                throw new BadCredentialsException("验证码错误");
            }
        }
    }
}
```

### 服务器端 Spring Security 配置

增加配置类 SecurityConfig 继承 WebSecurityConfigurerAdapter 抽象类，增加 @EnableWebSecurity 标注开启 Spring Security 配置功能，增加 @EnableGlobalMethodSecurity(prePostEnabled = true) 标注并将 prePostEnabled = true 说明 restfull 请求的方法上开启授权功能。

```java
...
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

	@Autowired
	public UserDetailsService userDetailsService;

    /**
     * 加密方式
     */
	@Bean
	public BCryptPasswordEncoder bCryptPasswordEncoder() {
		return new BCryptPasswordEncoder();
	}

	/**
	 * 全局用户信息
	 * 
	 * @param auth 认证管理
	 * @throws Exception 用户认证异常信息
	 */
	@Autowired
	public void globalUserDetails(AuthenticationManagerBuilder auth) throws Exception {
		auth.userDetailsService(userDetailsService).passwordEncoder(bCryptPasswordEncoder());
	}

	/**
	 * 认证管理
	 * 
	 * @return 认证管理对象
	 * @throws Exception 认证异常信息
	 */
	@Override
	@Bean
	public AuthenticationManager authenticationManagerBean() throws Exception {
		return super.authenticationManagerBean();
	}

	/**
	 * http安全配置
	 * 
	 * @param http http安全对象
	 * @throws Exception http安全异常信息
	 */
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.authorizeRequests().antMatchers(PermitAllUrl.permitAllUrl()).permitAll().anyRequest().authenticated().and()
				.httpBasic().and().csrf().disable();
	}

}
...
```

### 服务器端资源配置

增加了配置类 ResourceServerConfig 继承 ResourceServerConfigurerAdapter 类，增加 @EnableResourceServer 标注开启资源配置功能，增加 @Configuration 标注说明配置类。

**从源码分析中可以看出先进行认证（Authentication），再进行授权（Authorization）**

```java
...
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

	@Override
	public void configure(HttpSecurity http) throws Exception {
		http.requestMatcher(new OAuth2RequestedMatcher()).authorizeRequests().antMatchers(PermitAllUrl.permitAllUrl())
				.permitAll().anyRequest().authenticated();
	}

	/**
	 * 判断来源请求是否包含oauth2授权信息
	 */
	private static class OAuth2RequestedMatcher implements RequestMatcher {
		@Override
		public boolean matches(HttpServletRequest request) {
			// 请求参数中包含access_token参数
			if (request.getParameter(OAuth2AccessToken.ACCESS_TOKEN) != null) {
				return true;
			}

			// 头部的Authorization值以Bearer开头
			String auth = request.getHeader("Authorization");
			if (auth != null) {
				if (auth.startsWith(OAuth2AccessToken.BEARER_TYPE)) {
					return true;
				}
			}

			return false;
		}
	}

}
...
```

### 服务器开启 Session 共享

增加配置类 SessionConfig，增加 @EnableRedisHttpSession 标注开启 Session 共享功能。

```java
...
@EnableRedisHttpSession
public class SessionConfig {

}
```

### 客户端资源配置

增加配置类 ResourceServerConfig 继承 ResourceServerConfigurerAdapter 类，增加 @EnableResourceServer 标注开启资源配置功能，增加 @EnableWebSecurity 标注开启 Spring Security 配置功能，增加  @EnableGlobalMethodSecurity(prePostEnabled = true) 标注并将 prePostEnabled = true 说明 restfull 请求的方法上开启授权功能。

```java
...
@EnableResourceServer
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    /**
     * /*-anon/** 说明是微服务内部调用，外网网关不能调用
     */
	@Override
	public void configure(HttpSecurity http) throws Exception {
		http.csrf().disable().exceptionHandling()
				.authenticationEntryPoint(
						(request, response, authException) -> response.sendError(HttpServletResponse.SC_UNAUTHORIZED))
				.and().authorizeRequests().antMatchers(PermitAllUrl.permitAllUrl("/users-anon/**", "/wechat/**")).permitAll()
				.anyRequest().authenticated().and().httpBasic();
	}

	@Bean
	public BCryptPasswordEncoder bCryptPasswordEncoder() {
		return new BCryptPasswordEncoder();
	}

}    
```

### 客户端开启 Session 共享

```java
...
@EnableRedisHttpSession
public class SessionConfig {

}    
```



