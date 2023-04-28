# Spring Security认证与授权的原理

### 一、概念：

Spring Security所解决的问题就是安全访问控制，而安全访问控制功能其实就是所有进入系统的请求进行拦截，校验每一个请求是否能够访问它所期望的资源，可以通过Filter或AOP等技术来实现，Spring Security对web资源的保护是靠Filter来实现的。

当初始化Spring Security时，会创建一个名为SpringSecurityFilterChain的servlet过滤器，类型是org.springframework.security.web.FilterChainProxy（有doFilter方法），它实现了javax.servlet.Filter，因此外部请求的类都会经过此类。如图：

![img](https://img-blog.csdnimg.cn/20200503191744839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

实际上，看源代码我们知道FilterChainProxy是一个代理，真正起作用的是FilterChainProxy中SecurityFilterChain所包含的各个Filter，同时 这些Filter作为Bean被Spring管理，它们是Spring Security核心，各有各的职责，但他们并不直接处理用户的认证，也不直接处理用户的授权，而是把它们交给了认证管理器（AuthenticationManager）和决策管理器（AccessDecisionManager）进行处理，下图是FilterChainProxy相关类的UML图示（直接网上找图）。

![img](https://img-blog.csdnimg.cn/20200514222157974.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)



不知道UML图的小伙伴们，可以去学一波这个姿势。

Spring Security功能的实现主要是由一系列过滤器链相互配合完成。

![img](https://img-blog.csdnimg.cn/20200503192522763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

（1）SecurityContextPersistenceFilter：这个Filter是整个拦截过程的入口和出口(也就是第一个和最后- 一个拦截器) , 会在请求开始时从配置好的SecurityContextRepository中获取SecurityContext ,然后把它设置给SecurityContextHolder.在请求完成后将SecurityContextHolder持有的SecurityContext再保存到配置好的SecurityContextRepository ,同时清除securityContextHolder所持有的SecurityContext ; .

（2）UsernamePasswordAuthenticationFilter：用于处理来自表单提交的认证。该表单必须提供对应的用户名和密码,其内部还有登录成功或失败后进行处理的AuthenticationSuccessHandler和AuthenticationFailureHandler ,这些都可以根据需求做相关改变;

（3）FilteSecurityInterceptor：是用于保护web资源的,使用AccessDecisionManager对当前用户进行授权访问,前面已经详细介绍过了;

（4）ExceptionTranslationFilter：ExceptionTranslationFilter能够捕获来自FilterChain 所有的异常,并进行处理。但是它只会处理两类异常:ExceptionTranslationFilter能够捕获来自FilterChain 所有的异常,并进行处理。但是它只会处理两类异常:
AuthenticationException和AccessDeniedException ,其它的异常它会继续抛出。

### 二、认证流程：

![img](https://img-blog.csdnimg.cn/20200503193813635.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)


（1）用户提交用户名和密码被UsernamePasswordAuthenticationFilter获取到，然后请求的信息被封装为Authentication的实现类UsernamePasswordAuthenticationToken对象。我们来看一下这个源码：UsernamePasswordAuthenticationFilter类的attemptAuthentication方法。

```
public class UsernamePasswordAuthenticationFilter extends
		AbstractAuthenticationProcessingFilter {
	
	public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
	public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";
 
	private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
	private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
	private boolean postOnly = true;
 
	
 
	public UsernamePasswordAuthenticationFilter() {
		super(new AntPathRequestMatcher("/login", "POST"));
	}
 
	//设置Authentication
	public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}
        //获取参数
		String username = obtainUsername(request);
		String password = obtainPassword(request);
 
		if (username == null) {
			username = "";
		}
 
		if (password == null) {
			password = "";
		}
 
		username = username.trim();
        //创建Authentication
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);
 
		//设置主机地址和sessionId
		setDetails(request, authRequest);
        //通过这个方法去找到AuthenticationManager认证。
		return this.getAuthenticationManager().authenticate(authRequest);
	}
 
	
	@Nullable
	protected String obtainPassword(HttpServletRequest request) {
		return request.getParameter(passwordParameter);
	}
 
	
	@Nullable
	protected String obtainUsername(HttpServletRequest request) {
		return request.getParameter(usernameParameter);
	}
	//设置主机地址和sessionId
	protected void setDetails(HttpServletRequest request,
			UsernamePasswordAuthenticationToken authRequest) {
		authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
	}
}
```



（2）将Authentication（也就是上面的authRequest）交给AuthenticationManager去认证。之后我们就到了AuthenticationManager的子类ProviderManager这个类。经过辗转到DaoAuthenticationProvider的retrieveUser方法。通过loadUserByUsername找到对应的用户信息，实际上就是通过UserDetailService来办到的。

```
protected final UserDetails retrieveUser(String username,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
		prepareTimingAttackProtection();
		try {
//查询对应的用户信息
			UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
			if (loadedUser == null) {
				throw new InternalAuthenticationServiceException(
						"UserDetailsService returned null, which is an interface contract violation");
			}
//返回
			return loadedUser;
		}
		catch (UsernameNotFoundException ex) {
			mitigateAgainstTimingAttack(authentication);
			throw ex;
		}
		catch (InternalAuthenticationServiceException ex) {
			throw ex;
		}
		catch (Exception ex) {
			throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
		}
	}
```

![img](https://img-blog.csdnimg.cn/2020051423151314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

（3）核对认证：

```
protected void additionalAuthenticationChecks(UserDetails userDetails,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
//如果用户没输入密码，直接抛出异常！
		if (authentication.getCredentials() == null) {
			logger.debug("Authentication failed: no credentials provided");
 
			throw new BadCredentialsException(messages.getMessage(
					"AbstractUserDetailsAuthenticationProvider.badCredentials",
					"Bad credentials"));
		}
//获取用户的密码
		String presentedPassword = authentication.getCredentials().toString();
//用编码器去匹配
		if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			logger.debug("Authentication failed: password does not match stored value");
 
			throw new BadCredentialsException(messages.getMessage(
					"AbstractUserDetailsAuthenticationProvider.badCredentials",
					"Bad credentials"));
		}
	}
```

只要不抛出异常就是正确了。

（4）重新封装Authentication返回UsernamePasswordAuthenticationFilter。

![img](https://img-blog.csdnimg.cn/20200514232459603.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

（5）封装到上下文（AbstractAuthenticationProcessingFilter）：

![img](https://img-blog.csdnimg.cn/20200514233644210.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

我们再来看一下这里涉及的类：

（1）AuthenticationProvider ： AuthenticationManager委托这个接口的实现类来处理认证。

```
public interface AuthenticationProvider { 
    Authentication authenticate(Authentication authentication) throws         
    AuthenticationException; boolean supports(Class<?> var1); 
}
```

authenticate () 方法定义了 认证的实现过程 ，它的参数是一个 Authentication ，里面包含了登录用户所提交的用
户、密码等。而返回值也是一个 Authentication ，这个 Authentication 则是在认证成功后，将用户的权限及其他信
息重新组装后生成。

Spring Security 中维护着一个 List<AuthenticationProvider> 列表，存放多种认证方式，不同的认证方式使用不
同的 AuthenticationProvider 。如使用用户名密码登录时，使用 AuthenticationProvider1 ，短信登录时使用
AuthenticationProvider2。

每个 AuthenticationProvider 需要实现 supports （） 方法来表明自己支持的认证方式，如我们使用表单方式认证，
在提交请求时 Spring Security 会生成 UsernamePasswordAuthenticationToken ，它是一个 Authentication ，里面
封装着用户提交的用户名、密码信息。而对应的，哪个 AuthenticationProvider 来处理它。

```
//DaoAuthenticationProvider的基类AbstractUserDetailsAuthenticationProvider
public boolean supports(Class<?> authentication) { 
    return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication); 
}
```


也就是说当 web 表单提交用户名密码时， Spring Security 由 DaoAuthenticationProvider 处理。

（1）Authentication： 认证信息结构

```
public interface Authentication extends Principal, Serializable {
    //权限信息  
    Collection<? extends GrantedAuthority> getAuthorities(); 
    //密码
    Object getCredentials(); 
    //细节信息:ip，sessionId
    Object getDetails(); 
    //账号
    Object getPrincipal(); 
    boolean isAuthenticated(); void setAuthenticated(boolean var1) throws IllegalArgumentException; 
}
```

通过上面给大家debug的时候，不知道大家发现没有，principal这个字段在封装请求信息的时候是username，但是在认证通过后返回信息的时候就封装住了账号、密码以及权限等信息。

（3）UserDetailsService：

```
public interface UserDetailsService { 
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException; 
}
```

它的作用就是从数据库或者内存中查询出账号密码，封装成UserDetails，然后返回给我们DaoAuthenticationProvider，进行认证。我们可以实现一个类去继承这个Service，然后从数据库中查询出信息。之前在上篇文章中是用的内存的方式@Bean注入到IOC中了。

（4）UserDetails



```
public interface UserDetails extends Serializable { 
//权限
    Collection<? extends GrantedAuthority> getAuthorities(); 
//用户名
    String getPassword(); 
//密码
    String getUsername(); 
//
    boolean isAccountNonExpired(); 
    boolean isAccountNonLocked(); 
    boolean isCredentialsNonExpired(); 
    boolean isEnabled(); 
}
```


我们来扩展一下UserDetailService，从数据库中查询出信息：

```
@Service
public class SpringDataUserDetailService implements UserDetailsService {

    @Autowired
    private IUserService userService;
    @Autowired
    private PermissionMapper permissionMapper;
     
    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        //查询用户信息
        UserAdmin userAmin = userService.selectUserByUsername(s);
        log.info("user => {}",userAmin);
        if(userAmin == null){
            return null;
        }
        //查询权限
        List<String> list =  permissionMapper.selectPermissionByUser(userAmin.getId());
        String[] arr = new String[list.size()];
        list.toArray(arr);
        UserDetails user = User.withUsername(userAmin.getUsername()).password(userAmin.getPassword()).authorities(arr).build();
        return user;
    }

}
```

查询对应的用户信息和权限！！！

（4）PasswordEncoder：

DaoAuthenticationProvider 认证处理器通过 UserDetailsService 获取到 UserDetails 后，它是如何与请求 Authentication中的密码做对比呢？ 在这里Spring Security 为了适应多种多样的加密类型，又做了抽象， DaoAuthenticationProvider 通过 PasswordEncoder接口的 matches 方法进行密码的对比，而具体的密码对比细节取决于实现

```
public interface PasswordEncoder { 
    String encode(CharSequence var1); 
    boolean matches(CharSequence var1, String var2); 
    default boolean upgradeEncoding(String encodedPassword) { return false; } 
}
```

而 Spring Security 提供很多内置的 PasswordEncoder ，能够开箱即用，使用某种 PasswordEncoder 只需要进行如
下声明即可，如下

```
@Bean 
public PasswordEncoder passwordEncoder() { 
    return NoOpPasswordEncoder.getInstance(); 
}
```

NoOpPasswordEncoder 采用字符串匹配方法，不对密码进行加密比较处理，密码比较流程如下：

（1）用户输入密码（明文 ）
（2）DaoAuthenticationProvider 获取 UserDetails （其中存储了用户的正确密码）
（3）DaoAuthenticationProvider 使用 PasswordEncoder 对输入的密码和正确的密码进行校验，密码一致则校验通
过，否则校验失败。 NoOpPasswordEncoder 的校验规则拿 输入的密码和 UserDetails 中的正确密码进行字符串比较，字符串内容一致 则校验通过，否则 校验失败。

三、授权流程：
Spring Security 可以通过 http.authorizeRequests() 对 web 请求进行授权保护。 Spring Security使用标准 Filter 建立了对 web 请求的拦截，最终实现对资源的授权访问。

![img](https://img-blog.csdnimg.cn/20200503214152166.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

分析授权流程：
（1） 拦截请求： 已认证用户访问受保护的 web 资源将被 SecurityFilterChain 中的 FilterSecurityInterceptor 的子 类拦截。

 ![img](https://img-blog.csdnimg.cn/20200515195318922.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20200515195732146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)


这里面1就是授权的具体步骤，2是如果授权通过执行的真正的业务，3是后续处理。

（2）获取资源访问策略：FilterSecurityInterceptor 会从 SecurityMetadataSource 的子类
DefaultFilterInvocationSecurityMetadataSource 获取要访问当前资源所需要的权限 Collection<ConfigAttribute>。 SecurityMetadataSource 其实就是读取访问策略的抽象，而读取的内容，其实就是我们配置的访问规则， 读
取访问策略如：

```
 protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .authorizeRequests()
                .antMatchers("/r/r1").hasAuthority("r1")
                .antMatchers("/r/r2").hasAuthority("r2")
                ....
    }
```

进入AbstractSecurityInterceptor的beforeInvocation方法，获取对应的需要的权限：

![img](https://img-blog.csdnimg.cn/20200515195959248.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20200515201311361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20200515201431315.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)


p1和p2是用户现在拥有的权限。r1是要求的权限

（3）授权决策：FilterSecurityInterceptor 会调用 AccessDecisionManager 进行授权决策，若决策通过，则允许访问资
源，否则将禁止访问。

![img](https://img-blog.csdnimg.cn/20200515200038401.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)


稍后我们会具体的介绍都有哪些决策方式，这里我们以默认的为例AffirmativeBased是AccessDecisionManager子类

![img](https://img-blog.csdnimg.cn/20200515200254950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20200515201531678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDU4ODQ5NQ==,size_16,color_FFFFFF,t_70)

result就是决策投票的结果，如果是1代表赞成、-1代表反对、0代表弃权。如果通过，那么用户就可以访问到对应的资源。

三种授权决策 ：
AccessDecisionManager 采用 投票 的方式来确定是否能够访问受保护资源。

```
public interface AccessDecisionVoter<S> { 
    int ACCESS_GRANTED = 1; 
    int ACCESS_ABSTAIN = 0; 
    int ACCESS_DENIED = ‐1; 
    boolean supports(ConfigAttribute var1);     
    boolean supports(Class<?> var1); 
    int vote(Authentication var1, S var2, Collection<ConfigAttribute> var3); 
}
```

vote() 方法的返回结果会是 AccessDecisionVoter 中定义的三个常量之一。 ACCESS_GRANTED 表示同意， ACCESS_DENIED表示拒绝， ACCESS_ABSTAIN 表示弃权。如果一个 AccessDecisionVoter 不能判定当前 Authentication是否拥有访问对应受保护对象的权限，则其 vote() 方法的返回值应当为弃权 ACCESS_ABSTAIN 。

三个实现类：

1、AffiffiffirmativeBased：

（1）只要有AccessDecisionVoter的投票为ACCESS_GRANTED则同意用户进行访问；

（2）如果全部弃权也表示通过；

（3）如果没有一个人投赞成票，但是有人投反对票，则将抛出AccessDeniedException。

2、 ConsensusBased：
（1）如果赞成票多于反对票则表示通过。

（2）反过来，如果反对票多于赞成票则将抛出AccessDeniedException。

（3）如果赞成票与反对票相同且不等于0，并且属性allowIfEqualGrantedDeniedDecisions的值为true，则表 示通过，否则将抛出异常AccessDeniedException。参数allowIfEqualGrantedDeniedDecisions的值默认为true。

（4）如果所有的AccessDecisionVoter都弃权了，则将视参数allowIfAllAbstainDecisions的值而定，如果该值 为true则表示通过，否则将抛出异常AccessDeniedException。参数allowIfAllAbstainDecisions的值默认为false。

3、 UnanimousBased
逻辑与另外两种实现有点不一样，另外两种会一次性把受保护对象的配置属性全部传递 给AccessDecisionVoter进行投票，而UnanimousBased会一次只传递一个ConfifigAttribute给 AccessDecisionVoter进行投票。这也就意味着如果我们的AccessDecisionVoter的逻辑是只要传递进来的 ConfifigAttribute中有一个能够匹配则投赞成票，但是放到UnanimousBased中其投票结果就不一定是赞成了。 UnanimousBased的逻辑具体来说是这样的：

（1）如果受保护对象配置的某一个ConfifigAttribute被任意的AccessDecisionVoter反对了，则将抛出
AccessDeniedException。

（2）如果没有反对票，但是有赞成票，则表示通过。

（3）如果全部弃权了，则将视参数allowIfAllAbstainDecisions的值而定，true则通过，false则抛出
AccessDeniedException