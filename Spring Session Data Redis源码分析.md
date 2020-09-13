> 白菜Java自习室 涵盖核心知识

> Spring-Session-Data-Redis的版本2.2.x，SpringBoot整合

# 概要

@EnableRedisHttpSession注解位于spring-session项目的spring-session-data-redis模块的org.springframework.session.data.redis.config.annotation.web.http包中，是主要的初始化配置注解。

# @EnableRedisHttpSession

1. @EnableRedisHttpSession 注解定义时, 使用@Configuration 修饰, 说明是@EnableRedisHttpSession 修饰的类, 会被Spring 当做配置类进行解析;
2. @EnableRedisHttpSession 中使用@Import 注解导入了RedisHttpSessionConfiguration.class, 那么在应用启动时会向spring 容器中注入RedisHttpSessionConfiguration 实例。

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(RedisHttpSessionConfiguration.class)
@Configuration(proxyBeanMethods = false)
public @interface EnableRedisHttpSession {

	int maxInactiveIntervalInSeconds() default MapSession.DEFAULT_MAX_INACTIVE_INTERVAL_SECONDS;

	String redisNamespace() default RedisIndexedSessionRepository.DEFAULT_NAMESPACE;

	@Deprecated
	RedisFlushMode redisFlushMode() default RedisFlushMode.ON_SAVE;

	FlushMode flushMode() default FlushMode.ON_SAVE;

	String cleanupCron() default RedisHttpSessionConfiguration.DEFAULT_CLEANUP_CRON;

	SaveMode saveMode() default SaveMode.ON_SET_ATTRIBUTE;
	
}
```

# RedisHttpSessionConfiguration

1. RedisHttpSessionConfiguration 本身是一个Spring 配置类, 会向spring 容器注册sessionRepository, redisMessageListenerContainer 等实例;

```
@Configuration(proxyBeanMethods = false)
public class RedisHttpSessionConfiguration extends SpringHttpSessionConfiguration
		implements BeanClassLoaderAware, EmbeddedValueResolverAware, ImportAware {
	// ......

	@Override
	public void setImportMetadata(AnnotationMetadata importMetadata) {
		Map<String, Object> attributeMap = importMetadata
				.getAnnotationAttributes(EnableRedisHttpSession.class.getName());
		AnnotationAttributes attributes = AnnotationAttributes.fromMap(attributeMap);
		this.maxInactiveIntervalInSeconds = attributes.getNumber("maxInactiveIntervalInSeconds");
		String redisNamespaceValue = attributes.getString("redisNamespace");
		if (StringUtils.hasText(redisNamespaceValue)) {
			this.redisNamespace = this.embeddedValueResolver.resolveStringValue(redisNamespaceValue);
		}
		FlushMode flushMode = attributes.getEnum("flushMode");
		RedisFlushMode redisFlushMode = attributes.getEnum("redisFlushMode");
		if (flushMode == FlushMode.ON_SAVE && redisFlushMode != RedisFlushMode.ON_SAVE) {
			flushMode = redisFlushMode.getFlushMode();
		}
		this.flushMode = flushMode;
		this.saveMode = attributes.getEnum("saveMode");
		String cleanupCron = attributes.getString("cleanupCron");
		if (StringUtils.hasText(cleanupCron)) {
			this.cleanupCron = cleanupCron;
		}
	}

	// ......
}
```

2. RedisHttpSessionConfiguration 会注册Redis 消息监听器容器RedisMessageListenerContainer, 并将RedisIndexedSessionRepository 作为Redis 消息订阅的监听器, 因为它实现了MessageListener接口。当Redis 中key 过期或销毁时, 会通知将RedisIndexedSessionRepository 调用其onMessage 方法来处理消息;

```
@Configuration(proxyBeanMethods = false)
public class RedisHttpSessionConfiguration extends SpringHttpSessionConfiguration
		implements BeanClassLoaderAware, EmbeddedValueResolverAware, ImportAware {
	// ......

	@Bean
	public RedisIndexedSessionRepository sessionRepository() {
		RedisTemplate<Object, Object> redisTemplate = createRedisTemplate();
		RedisIndexedSessionRepository sessionRepository = new RedisIndexedSessionRepository(redisTemplate);
		sessionRepository.setApplicationEventPublisher(this.applicationEventPublisher);
		if (this.indexResolver != null) {
			sessionRepository.setIndexResolver(this.indexResolver);
		}
		if (this.defaultRedisSerializer != null) {
			sessionRepository.setDefaultSerializer(this.defaultRedisSerializer);
		}
		sessionRepository.setDefaultMaxInactiveInterval(this.maxInactiveIntervalInSeconds);
		if (StringUtils.hasText(this.redisNamespace)) {
			sessionRepository.setRedisKeyNamespace(this.redisNamespace);
		}
		sessionRepository.setFlushMode(this.flushMode);
		sessionRepository.setSaveMode(this.saveMode);
		int database = resolveDatabase();
		sessionRepository.setDatabase(database);
		this.sessionRepositoryCustomizers
				.forEach((sessionRepositoryCustomizer) -> sessionRepositoryCustomizer.customize(sessionRepository));
		return sessionRepository;
	}
	
	@Bean
	public RedisMessageListenerContainer springSessionRedisMessageListenerContainer(
			RedisIndexedSessionRepository sessionRepository) {
		RedisMessageListenerContainer container = new RedisMessageListenerContainer();
		container.setConnectionFactory(this.redisConnectionFactory);
		if (this.redisTaskExecutor != null) {
			container.setTaskExecutor(this.redisTaskExecutor);
		}
		if (this.redisSubscriptionExecutor != null) {
			container.setSubscriptionExecutor(this.redisSubscriptionExecutor);
		}
		container.addMessageListener(sessionRepository,
				Arrays.asList(new ChannelTopic(sessionRepository.getSessionDeletedChannel()),
						new ChannelTopic(sessionRepository.getSessionExpiredChannel())));
		container.addMessageListener(sessionRepository,
				Collections.singletonList(new PatternTopic(sessionRepository.getSessionCreatedChannelPrefix() + "*")));
		return container;
	}

	@Bean
	public InitializingBean enableRedisKeyspaceNotificationsInitializer() {
		return new EnableRedisKeyspaceNotificationsInitializer(this.redisConnectionFactory, this.configureRedisAction);
	}

	@Autowired(required = false)
	public void setConfigureRedisAction(ConfigureRedisAction configureRedisAction) {
		this.configureRedisAction = configureRedisAction;
	}

	@Autowired
	public void setRedisConnectionFactory(
			@SpringSessionRedisConnectionFactory ObjectProvider<RedisConnectionFactory> springSessionRedisConnectionFactory,
			ObjectProvider<RedisConnectionFactory> redisConnectionFactory) {
		RedisConnectionFactory redisConnectionFactoryToUse = springSessionRedisConnectionFactory.getIfAvailable();
		if (redisConnectionFactoryToUse == null) {
			redisConnectionFactoryToUse = redisConnectionFactory.getObject();
		}
		this.redisConnectionFactory = redisConnectionFactoryToUse;
	}

	@Autowired(required = false)
	@Qualifier("springSessionDefaultRedisSerializer")
	public void setDefaultRedisSerializer(RedisSerializer<Object> defaultRedisSerializer) {
		this.defaultRedisSerializer = defaultRedisSerializer;
	}

	@Autowired
	public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
		this.applicationEventPublisher = applicationEventPublisher;
	}

	@Autowired(required = false)
	public void setIndexResolver(IndexResolver<Session> indexResolver) {
		this.indexResolver = indexResolver;
	}

	@Autowired(required = false)
	@Qualifier("springSessionRedisTaskExecutor")
	public void setRedisTaskExecutor(Executor redisTaskExecutor) {
		this.redisTaskExecutor = redisTaskExecutor;
	}

	@Autowired(required = false)
	@Qualifier("springSessionRedisSubscriptionExecutor")
	public void setRedisSubscriptionExecutor(Executor redisSubscriptionExecutor) {
		this.redisSubscriptionExecutor = redisSubscriptionExecutor;
	}

	@Autowired(required = false)
	public void setSessionRepositoryCustomizer(
			ObjectProvider<SessionRepositoryCustomizer<RedisIndexedSessionRepository>> sessionRepositoryCustomizers) {
		this.sessionRepositoryCustomizers = sessionRepositoryCustomizers.orderedStream().collect(Collectors.toList());
	}
	
	// ......
	
        private RedisTemplate<Object, Object> createRedisTemplate() {
		RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
		redisTemplate.setKeySerializer(new StringRedisSerializer());
		redisTemplate.setHashKeySerializer(new StringRedisSerializer());
		if (this.defaultRedisSerializer != null) {
			redisTemplate.setDefaultSerializer(this.defaultRedisSerializer);
		}
		redisTemplate.setConnectionFactory(this.redisConnectionFactory);
		redisTemplate.setBeanClassLoader(this.classLoader);
		redisTemplate.afterPropertiesSet();
		return redisTemplate;
	}

	private int resolveDatabase() {
		if (ClassUtils.isPresent("io.lettuce.core.RedisClient", null)
				&& this.redisConnectionFactory instanceof LettuceConnectionFactory) {
			return ((LettuceConnectionFactory) this.redisConnectionFactory).getDatabase();
		}
		if (ClassUtils.isPresent("redis.clients.jedis.Jedis", null)
				&& this.redisConnectionFactory instanceof JedisConnectionFactory) {
			return ((JedisConnectionFactory) this.redisConnectionFactory).getDatabase();
		}
		return RedisIndexedSessionRepository.DEFAULT_DATABASE;
	}

}
```

两个内部的配置类EnableRedisKeyspaceNotificationsInitializer和SessionCleanupConfiguration:

```
static class EnableRedisKeyspaceNotificationsInitializer implements InitializingBean {

	private final RedisConnectionFactory connectionFactory;

	private ConfigureRedisAction configure;

	EnableRedisKeyspaceNotificationsInitializer(RedisConnectionFactory connectionFactory,
			ConfigureRedisAction configure) {
		this.connectionFactory = connectionFactory;
		this.configure = configure;
	}

	@Override
	public void afterPropertiesSet() {
		if (this.configure == ConfigureRedisAction.NO_OP) {
			return;
		}
		RedisConnection connection = this.connectionFactory.getConnection();
		try {
			this.configure.configure(connection);
		}
		finally {
			try {
				connection.close();
			}
			catch (Exception ex) {
				LogFactory.getLog(getClass()).error("Error closing RedisConnection", ex);
			}
		}
	}

}
```

```
@EnableScheduling
@Configuration(proxyBeanMethods = false)
class SessionCleanupConfiguration implements SchedulingConfigurer {

	private final RedisIndexedSessionRepository sessionRepository;

	SessionCleanupConfiguration(RedisIndexedSessionRepository sessionRepository) {
		this.sessionRepository = sessionRepository;
	}

	@Override
	public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
		taskRegistrar.addCronTask(this.sessionRepository::cleanupExpiredSessions,
				RedisHttpSessionConfiguration.this.cleanupCron);
	}

}
```

## RedisMessageListenerContainer

RedisMessageListenerContainer实现了InitializingBean, DisposableBean, BeanNameAware, SmartLifecycle几个接口。

1. InitializingBean：主要实现afterPropertiesSet方法，来定义spring设置完properties后进行的处理，在spring init这个bean时候会被调用;
2. DisposableBean：实现destroy方法，在spring销毁bean时会调用;
3. BeanNameAware：实现setBeanName方法来为bean进行取名，在RedisMessageListenerContainer中该name被用于内部线程的线程名;
4. SmartLifecycle：spring的bean生命周期类，spring会调用start，stop等操作来完成。

```
public class RedisMessageListenerContainer implements InitializingBean, DisposableBean, BeanNameAware, SmartLifecycle {
   
    public RedisMessageListenerContainer() {
        this.initWait = TimeUnit.SECONDS.toMillis(5L);
        this.monitor = new Object();
        this.running = false;
        this.initialized = false;
        this.listening = false;
        this.manageExecutor = false;
        this.patternMapping = new ConcurrentHashMap();
        this.channelMapping = new ConcurrentHashMap();
        this.listenerTopics = new ConcurrentHashMap();
        this.subscriptionTask = new RedisMessageListenerContainer.SubscriptionTask();
        this.serializer = new StringRedisSerializer();
        this.recoveryInterval = 5000L;
        this.maxSubscriptionRegistrationWaitingTime = 2000L;
    }

    // ......
}
```

RedisMessageListenerContainer的启动顺序：

1. Spring先完成对于bean属性的set，其中包含listener map的set操作。
2. 调用afterPropertiesSet方法，此方法构造了一个线程池来跑监听线程。
```
public void afterPropertiesSet() {
	if (taskExecutor == null) {
		manageExecutor = true;
		taskExecutor = createDefaultTaskExecutor();
	}

	if (subscriptionExecutor == null) {
		subscriptionExecutor = taskExecutor;
	}

	initialized = true;
}
```

3. spring调用start方法来开启这个bean。
```
public void start() {
	if (!running) {
		running = true;
		// wait for the subscription to start before returning
		// technically speaking we can only be notified right before the subscription starts
		synchronized (monitor) {
			lazyListen();
			if (listening) {
				try {
					// wait up to 5 seconds for Subscription thread
					monitor.wait(initWait);
				} catch (InterruptedException e) {
					// stop waiting
					Thread.currentThread().interrupt();
					running = false;
					return;
				}
			}
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Started RedisMessageListenerContainer");
		}
	}
}
```

4. 最重要的一步lazyListen()方法的调用，构造一个SubscriptionTask，并且提交给第二步afterPropertiesSet方法中创建的线程池来执行。
```
private void lazyListen() {
	boolean debug = logger.isDebugEnabled();
	boolean started = false;

	if (isRunning()) {
		if (!listening) {
			synchronized (monitor) {
				if (!listening) {
					if (channelMapping.size() > 0 || patternMapping.size() > 0) {
						subscriptionExecutor.execute(subscriptionTask);
						listening = true;
						started = true;
					}
				}
			}
			if (debug) {
				if (started) {
					logger.debug("Started listening for Redis messages");
				} else {
					logger.debug("Postpone listening for Redis messages until actual listeners are added");
				}
			}
		}
	}
}
```

## RedisIndexedSessionRepository

RedisIndexedSessionRepository实现了MessageListener接口。当Redis 中key 过期或销毁时, 会通知将RedisIndexedSessionRepository 调用其onMessage 方法来处理消息。

```
public class RedisIndexedSessionRepository
		implements FindByIndexNameSessionRepository<RedisIndexedSessionRepository.RedisSession>, MessageListener {
    // ......
    
	@Override
	public void onMessage(Message message, byte[] pattern) {
		byte[] messageChannel = message.getChannel();
		byte[] messageBody = message.getBody();

		String channel = new String(messageChannel);

		if (channel.startsWith(this.sessionCreatedChannelPrefix)) {
			// TODO: is this thread safe?
			@SuppressWarnings("unchecked")
			Map<Object, Object> loaded = (Map<Object, Object>) this.defaultSerializer.deserialize(message.getBody());
			handleCreated(loaded, channel);
			return;
		}

		String body = new String(messageBody);
		if (!body.startsWith(getExpiredKeyPrefix())) {
			return;
		}

		boolean isDeleted = channel.equals(this.sessionDeletedChannel);
		if (isDeleted || channel.equals(this.sessionExpiredChannel)) {
			int beginIndex = body.lastIndexOf(":") + 1;
			int endIndex = body.length();
			String sessionId = body.substring(beginIndex, endIndex);

			RedisSession session = getSession(sessionId, true);

			if (session == null) {
				logger.warn("Unable to publish SessionDestroyedEvent for session " + sessionId);
				return;
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Publishing SessionDestroyedEvent for session " + sessionId);
			}

			cleanupPrincipalIndex(session);

			if (isDeleted) {
				handleDeleted(session);
			}
			else {
				handleExpired(session);
			}
		}
	}
	
    // ......
}
```

RedisIndexedSessionRepository还实现了许多操作Redis Session的操作方法，如save(), findById(), findByIndexNameAndIndexValue(), deleteById() 等。

```
@Override
public void save(RedisSession session) {
	session.save();
	if (session.isNew) {
		String sessionCreatedKey = getSessionCreatedChannel(session.getId());
		this.sessionRedisOperations.convertAndSend(sessionCreatedKey, session.delta);
		session.isNew = false;
	}
}

public void cleanupExpiredSessions() {
	this.expirationPolicy.cleanExpiredSessions();
}

@Override
public RedisSession findById(String id) {
	return getSession(id, false);
}

@Override
public Map<String, RedisSession> findByIndexNameAndIndexValue(String indexName, String indexValue) {
	if (!PRINCIPAL_NAME_INDEX_NAME.equals(indexName)) {
		return Collections.emptyMap();
	}
	String principalKey = getPrincipalKey(indexValue);
	Set<Object> sessionIds = this.sessionRedisOperations.boundSetOps(principalKey).members();
	Map<String, RedisSession> sessions = new HashMap<>(sessionIds.size());
	for (Object id : sessionIds) {
		RedisSession session = findById((String) id);
		if (session != null) {
			sessions.put(session.getId(), session);
		}
	}
	return sessions;
}

private RedisSession getSession(String id, boolean allowExpired) {
	Map<Object, Object> entries = getSessionBoundHashOperations(id).entries();
	if (entries.isEmpty()) {
		return null;
	}
	MapSession loaded = loadSession(id, entries);
	if (!allowExpired && loaded.isExpired()) {
		return null;
	}
	RedisSession result = new RedisSession(loaded, false);
	result.originalLastAccessTime = loaded.getLastAccessedTime();
	return result;
}

private MapSession loadSession(String id, Map<Object, Object> entries) {
	MapSession loaded = new MapSession(id);
	for (Map.Entry<Object, Object> entry : entries.entrySet()) {
		String key = (String) entry.getKey();
		if (RedisSessionMapper.CREATION_TIME_KEY.equals(key)) {
			loaded.setCreationTime(Instant.ofEpochMilli((long) entry.getValue()));
		}
		else if (RedisSessionMapper.MAX_INACTIVE_INTERVAL_KEY.equals(key)) {
			loaded.setMaxInactiveInterval(Duration.ofSeconds((int) entry.getValue()));
		}
		else if (RedisSessionMapper.LAST_ACCESSED_TIME_KEY.equals(key)) {
			loaded.setLastAccessedTime(Instant.ofEpochMilli((long) entry.getValue()));
		}
		else if (key.startsWith(RedisSessionMapper.ATTRIBUTE_PREFIX)) {
			loaded.setAttribute(key.substring(RedisSessionMapper.ATTRIBUTE_PREFIX.length()), entry.getValue());
		}
	}
	return loaded;
}

@Override
public void deleteById(String sessionId) {
	RedisSession session = getSession(sessionId, true);
	if (session == null) {
		return;
	}

	cleanupPrincipalIndex(session);
	this.expirationPolicy.onDelete(session);

	String expireKey = getExpiredKey(session.getId());
	this.sessionRedisOperations.delete(expireKey);

	session.setMaxInactiveInterval(Duration.ZERO);
	save(session);
}

@Override
public RedisSession createSession() {
	MapSession cached = new MapSession();
	if (this.defaultMaxInactiveInterval != null) {
		cached.setMaxInactiveInterval(Duration.ofSeconds(this.defaultMaxInactiveInterval));
	}
	RedisSession session = new RedisSession(cached, true);
	session.flushImmediateIfNecessary();
	return session;
}
```

3. RedisHttpSessionConfiguration 继承了SpringHttpSessionConfiguration, 所以会继承SpringHttpSessionConfiguration中的公有方法。

# SpringHttpSessionConfiguration

1. SpringHttpSessionConfiguration 会将自定义的所有SessionListener 封装为一个SessionEventHttpSessionListenerAdapter;
2. SpringHttpSessionConfiguration 会初始化一个最核心的组件SessionRepositoryFilter, 该过滤器会拦截所有的http 请求, 解析并处理session。

```
@Configuration(proxyBeanMethods = false)
public class SpringHttpSessionConfiguration implements ApplicationContextAware {

	// ......

	@PostConstruct
	public void init() {
		CookieSerializer cookieSerializer = (this.cookieSerializer != null) ? this.cookieSerializer
				: createDefaultCookieSerializer();
		this.defaultHttpSessionIdResolver.setCookieSerializer(cookieSerializer);
	}

	@Bean
	public SessionEventHttpSessionListenerAdapter sessionEventHttpSessionListenerAdapter() {
		return new SessionEventHttpSessionListenerAdapter(this.httpSessionListeners);
	}

	@Bean
	public <S extends Session> SessionRepositoryFilter<? extends Session> springSessionRepositoryFilter(
			SessionRepository<S> sessionRepository) {
		SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter<>(sessionRepository);
		sessionRepositoryFilter.setHttpSessionIdResolver(this.httpSessionIdResolver);
		return sessionRepositoryFilter;
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		if (ClassUtils.isPresent("org.springframework.security.web.authentication.RememberMeServices", null)) {
			this.usesSpringSessionRememberMeServices = !ObjectUtils
					.isEmpty(applicationContext.getBeanNamesForType(SpringSessionRememberMeServices.class));
		}
	}

	@Autowired(required = false)
	public void setServletContext(ServletContext servletContext) {
		this.servletContext = servletContext;
	}

	@Autowired(required = false)
	public void setCookieSerializer(CookieSerializer cookieSerializer) {
		this.cookieSerializer = cookieSerializer;
	}

	@Autowired(required = false)
	public void setHttpSessionIdResolver(HttpSessionIdResolver httpSessionIdResolver) {
		this.httpSessionIdResolver = httpSessionIdResolver;
	}

	@Autowired(required = false)
	public void setHttpSessionListeners(List<HttpSessionListener> listeners) {
		this.httpSessionListeners = listeners;
	}

	private CookieSerializer createDefaultCookieSerializer() {
		DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
		if (this.servletContext != null) {
			SessionCookieConfig sessionCookieConfig = null;
			try {
				sessionCookieConfig = this.servletContext.getSessionCookieConfig();
			}
			catch (UnsupportedOperationException ex) {
				this.logger.warn("Unable to obtain SessionCookieConfig: " + ex.getMessage());
			}
			if (sessionCookieConfig != null) {
				if (sessionCookieConfig.getName() != null) {
					cookieSerializer.setCookieName(sessionCookieConfig.getName());
				}
				if (sessionCookieConfig.getDomain() != null) {
					cookieSerializer.setDomainName(sessionCookieConfig.getDomain());
				}
				if (sessionCookieConfig.getPath() != null) {
					cookieSerializer.setCookiePath(sessionCookieConfig.getPath());
				}
				if (sessionCookieConfig.getMaxAge() != -1) {
					cookieSerializer.setCookieMaxAge(sessionCookieConfig.getMaxAge());
				}
			}
		}
		if (this.usesSpringSessionRememberMeServices) {
			cookieSerializer.setRememberMeRequestAttribute(SpringSessionRememberMeServices.REMEMBER_ME_LOGIN_ATTR);
		}
		return cookieSerializer;
	}

}
```

## SessionRepositoryFilter

SessionRepositoryFilter的拦截过程：

1. 请求被DelegatingFilterProxy 拦截到，然后执行doFilter()方法，在doFilter()中找到执行的代理类。
```
public class DelegatingFilterProxy extends GenericFilterBean {

    // ......

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		// Lazily initialize the delegate if necessary.
		Filter delegateToUse = this.delegate;
		if (delegateToUse == null) {
			synchronized (this.delegateMonitor) {
				delegateToUse = this.delegate;
				if (delegateToUse == null) {
					WebApplicationContext wac = findWebApplicationContext();
					if (wac == null) {
						throw new IllegalStateException("No WebApplicationContext found: " +
								"no ContextLoaderListener or DispatcherServlet registered?");
					}
					delegateToUse = initDelegate(wac);
				}
				this.delegate = delegateToUse;
			}
		}

		// Let the delegate perform the actual doFilter operation.
		invokeDelegate(delegateToUse, request, response, filterChain);
	}

	protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
		String targetBeanName = getTargetBeanName();
		Assert.state(targetBeanName != null, "No target bean name set");
		Filter delegate = wac.getBean(targetBeanName, Filter.class);
		if (isTargetFilterLifecycle()) {
			delegate.init(getFilterConfig());
		}
		return delegate;
	}

	protected void invokeDelegate(
			Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		delegate.doFilter(request, response, filterChain);
	}

	protected void destroyDelegate(Filter delegate) {
		if (isTargetFilterLifecycle()) {
			delegate.destroy();
		}
	}
	
    // ......
}
```

2. OncePerRequestFilter 代理Filter执行doFilter()方法，然后调用抽象方法doFilterInternal()。
```
abstract class OncePerRequestFilter implements Filter {
    // ......
    
	@Override
	public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		if (!(request instanceof HttpServletRequest) || !(response instanceof HttpServletResponse)) {
			throw new ServletException("OncePerRequestFilter just supports HTTP requests");
		}
		HttpServletRequest httpRequest = (HttpServletRequest) request;
		HttpServletResponse httpResponse = (HttpServletResponse) response;
		String alreadyFilteredAttributeName = this.alreadyFilteredAttributeName;
		boolean hasAlreadyFilteredAttribute = request.getAttribute(alreadyFilteredAttributeName) != null;

		if (hasAlreadyFilteredAttribute) {
			if (DispatcherType.ERROR.equals(request.getDispatcherType())) {
				doFilterNestedErrorDispatch(httpRequest, httpResponse, filterChain);
				return;
			}
			// Proceed without invoking this filter...
			filterChain.doFilter(request, response);
		}
		else {
			// Do invoke this filter...
			request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
			try {
				doFilterInternal(httpRequest, httpResponse, filterChain);
			}
			finally {
				// Remove the "already filtered" request attribute for this request.
				request.removeAttribute(alreadyFilteredAttributeName);
			}
		}
	}
	
    // ......
}
```

3. SessionRepositoryFilter 继承了OncePerRequestFilter，实现了doFilterInternal()，这个方法一个封装一个wrappedRequest，通过执行commitSession()保存session信息到redis。
```
@Order(SessionRepositoryFilter.DEFAULT_ORDER)
public class SessionRepositoryFilter<S extends Session> extends OncePerRequestFilter {
    // ......
    
    	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);

		SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(request, response);
		SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(wrappedRequest,
				response);

		try {
			filterChain.doFilter(wrappedRequest, wrappedResponse);
		}
		finally {
			wrappedRequest.commitSession();
		}
	}
	
    // ......
}
```