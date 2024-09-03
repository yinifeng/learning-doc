# 一、spring-security

- `@EnableWebSecurity`启用spring-security自动配置

  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  @Documented
  @Import({ WebSecurityConfiguration.class, SpringWebMvcImportSelector.class, OAuth2ImportSelector.class,
  		HttpSecurityConfiguration.class })
  @EnableGlobalAuthentication
  @Configuration
  public @interface EnableWebSecurity {
  
  	/**
  	 * Controls debugging support for Spring Security. Default is false.
  	 * @return if true, enables debug support with Spring Security
  	 */
  	boolean debug() default false;
  
  }
  ```

  1、`HttpSecurityConfiguration#httpSecurity`自动装配原型的`HttpSecurity`，用于构建`SecurityFilterChain`

  2、`WebSecurityConfiguration#setFilterChainProxySecurityConfigurer`先依赖注入构建`WebSecurity`

  3、`WebSecurityConfiguration`依赖查找配置`WebSecurityConfiguration#webSecurityConfigurers`

  4、`WebSecurityConfiguration`依赖查找配置`WebSecurityConfiguration#securityFilterChains`

  5、`WebSecurityConfiguration`依赖查找配置`WebSecurityConfiguration#webSecurityCustomizers`，5.4的API对`WebSecurity`的扩展点

  6、`WebSecurityConfiguration#springSecurityFilterChain`自动装配`FilterChainProxy`,也就是`WebSecurity#build`

  7、`@EnableGlobalAuthentication`注解会导入`AuthenticationConfiguration`认证配置类，这个主要是装配`AuthenticationManager`认证管理Bean

  

- `HttpSecurity`用来构建`SecurityFilterChain`

  类继承关系图：

  ![](.\images\HttpSecurity.png)

  1、主要通过`SecurityBuilder#build`的抽象类控制只创建一个对象

  2、`HttpSecurity#performBuild`方法构建`DefaultSecurityFilterChain`对象

  ```java
  	@Override
  	protected DefaultSecurityFilterChain performBuild() {
  		ExpressionUrlAuthorizationConfigurer<?> expressionConfigurer = getConfigurer(
  				ExpressionUrlAuthorizationConfigurer.class);
  		AuthorizeHttpRequestsConfigurer<?> httpConfigurer = getConfigurer(AuthorizeHttpRequestsConfigurer.class);
  		boolean oneConfigurerPresent = expressionConfigurer == null ^ httpConfigurer == null;
  		Assert.state((expressionConfigurer == null && httpConfigurer == null) || oneConfigurerPresent,
  				"authorizeHttpRequests cannot be used in conjunction with authorizeRequests. Please select just one.");
  		this.filters.sort(OrderComparator.INSTANCE);
  		List<Filter> sortedFilters = new ArrayList<>(this.filters.size());
  		for (Filter filter : this.filters) {
  			sortedFilters.add(((OrderedFilter) filter).filter);
  		}
  		return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters);
  	}
  ```

  

  

- `WebSecurity`用来构建`Filter`主要是这个`FilterChainProxy`

  类继承关系图：

  ![](.\images\WebSecurity.png)

  1、主要通过`SecurityBuilder#build`的抽象类控制只创建一个对象

  2、`WebSecurity#performBuild`方法构建`FilterChainProxy`

  ```java
  	@Override
  	protected Filter performBuild() throws Exception {
  		Assert.state(!this.securityFilterChainBuilders.isEmpty(),
  				() -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
  						+ "Typically this is done by exposing a SecurityFilterChain bean. "
  						+ "More advanced users can invoke " + WebSecurity.class.getSimpleName()
  						+ ".addSecurityFilterChainBuilder directly");
  		int chainSize = this.ignoredRequests.size() + this.securityFilterChainBuilders.size();
  		List<SecurityFilterChain> securityFilterChains = new ArrayList<>(chainSize);
  		List<RequestMatcherEntry<List<WebInvocationPrivilegeEvaluator>>> requestMatcherPrivilegeEvaluatorsEntries = new ArrayList<>();
  		for (RequestMatcher ignoredRequest : this.ignoredRequests) {
  			WebSecurity.this.logger.warn("You are asking Spring Security to ignore " + ignoredRequest
  					+ ". This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead.");
  			SecurityFilterChain securityFilterChain = new DefaultSecurityFilterChain(ignoredRequest);
  			securityFilterChains.add(securityFilterChain);
  			requestMatcherPrivilegeEvaluatorsEntries
  					.add(getRequestMatcherPrivilegeEvaluatorsEntry(securityFilterChain));
  		}
  		for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : this.securityFilterChainBuilders) {
  			SecurityFilterChain securityFilterChain = securityFilterChainBuilder.build();
  			securityFilterChains.add(securityFilterChain);
  			requestMatcherPrivilegeEvaluatorsEntries
  					.add(getRequestMatcherPrivilegeEvaluatorsEntry(securityFilterChain));
  		}
  		if (this.privilegeEvaluator == null) {
  			this.privilegeEvaluator = new RequestMatcherDelegatingWebInvocationPrivilegeEvaluator(
  					requestMatcherPrivilegeEvaluatorsEntries);
  		}
  		FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
  		if (this.httpFirewall != null) {
  			filterChainProxy.setFirewall(this.httpFirewall);
  		}
  		if (this.requestRejectedHandler != null) {
  			filterChainProxy.setRequestRejectedHandler(this.requestRejectedHandler);
  		}
  		filterChainProxy.afterPropertiesSet();
  
  		Filter result = filterChainProxy;
  		if (this.debugEnabled) {
  			this.logger.warn("\n\n" + "********************************************************************\n"
  					+ "**********        Security debugging is enabled.       *************\n"
  					+ "**********    This may include sensitive information.  *************\n"
  					+ "**********      Do not use in a production system!     *************\n"
  					+ "********************************************************************\n\n");
  			result = new DebugFilter(filterChainProxy);
  		}
  		this.postBuildAction.run();
  		return result;
  	}
  ```

  

- spring-security执行流程

  1、`FilterChainProxy`入口`Filter`过滤器，执行方法`FilterChainProxy#doFilterInternal`

  2、`FilterChainProxy#doFilterInternal`方法，通过`HttpServletRequest`从`SecurityFilterChain`数组中匹配第一个`SecurityFilterChain`

  3、匹配的`SecurityFilterChain`取出`Filter`数组，创建一个`VirtualFilterChain`链条对象，一个一个的执行匹配的`Filter`

  