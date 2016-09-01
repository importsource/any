spring_cache

##缓存简介
缓存，我的理解是：让数据更接近于使用者；工作机制是：先从缓存中读取数据，如果没有再从慢速设备上读取实际数据（数据也会存入缓存）；缓存什么：那些经常读取且不经常修改的数据/那些昂贵（CPU/IO）的且对于相同的请求有相同的计算结果的数据。如CPU--L1/L2--内存--磁盘就是一个典型的例子，CPU需要数据时先从L1/L2中读取，如果没有到内存中找，如果还没有会到磁盘上找。还有如用过Maven的朋友都应该知道，我们找依赖的时候，先从本机仓库找，再从本地服务器仓库找，最后到远程仓库服务器找；还有如京东的物流为什么那么快？他们在各个地都有分仓库，如果该仓库有货物那么送货的速度是非常快的。
 
##缓存命中率
即从缓存中读取数据的次数 与 总读取次数的比率，命中率越高越好：
命中率 = 从缓存中读取次数 / (总读取次数[从缓存中读取次数 + 从慢速设备上读取的次数])
Miss率 = 没有从缓存中读取的次数 / (总读取次数[从缓存中读取次数 + 从慢速设备上读取的次数])
 
这是一个非常重要的监控指标，如果做缓存一定要健康这个指标来看缓存是否工作良好；
 
##缓存策略
Eviction policy
移除策略，即如果缓存满了，从缓存中移除数据的策略；常见的有LFU、LRU、FIFO：
FIFO（First In First Out）：先进先出算法，即先放入缓存的先被移除；
LRU（Least Recently Used）：最久未使用算法，使用时间距离现在最久的那个被移除；
LFU（Least Frequently Used）：最近最少使用算法，一定时间段内使用次数（频率）最少的那个被移除；
 
TTL（Time To Live ）
存活期，即从缓存中创建时间点开始直到它到期的一个时间段（不管在这个时间段内有没有访问都将过期）
 
TTI（Time To Idle）
空闲期，即一个数据多久没被访问将从缓存中移除的时间。
 
 
到此，基本了解了缓存的知识，在Java中，我们一般对调用方法进行缓存控制，比如我调用"findUserById(Long id)"，那么我应该在调用这个方法之前先从缓存中查找有没有，如果没有再掉该方法如从数据库加载用户，然后添加到缓存中，下次调用时将会从缓存中获取到数据。
 
自Spring 3.1起，提供了类似于@Transactional注解事务的注解Cache支持，且提供了Cache抽象；在此之前一般通过AOP实现；使用Spring ###Cache的好处：
提供基本的Cache抽象，方便切换各种底层Cache；
通过注解Cache可以实现类似于事务一样，缓存逻辑透明的应用到我们的业务代码上，且只需要更少的代码就可以完成；
提供事务回滚时也自动回滚缓存；
支持比较复杂的缓存逻辑；
 
对于Spring Cache抽象，主要从以下几个方面学习：
Cache API及默认提供的实现
Cache注解
实现复杂的Cache逻辑
 
Cache API及默认提供的实现
Spring提供的核心Cache接口： 
Java代码  收藏代码
package org.springframework.cache;  
  
public interface Cache {  
    String getName();  //缓存的名字  
    Object getNativeCache(); //得到底层使用的缓存，如Ehcache  
    ValueWrapper get(Object key); //根据key得到一个ValueWrapper，然后调用其get方法获取值  
    <T> T get(Object key, Class<T> type);//根据key，和value的类型直接获取value  
    void put(Object key, Object value);//往缓存放数据  
    void evict(Object key);//从缓存中移除key对应的缓存  
    void clear(); //清空缓存  
  
    interface ValueWrapper { //缓存值的Wrapper  
        Object get(); //得到真实的value  
        }  
}  
提供了缓存操作的读取/写入/移除方法；
 
 
默认提供了如下实现：
ConcurrentMapCache：使用java.util.concurrent.ConcurrentHashMap实现的Cache；
GuavaCache：对Guava com.google.common.cache.Cache进行的Wrapper，需要Google Guava 12.0或更高版本，@since spring 4；
EhCacheCache：使用Ehcache实现
JCacheCache：对javax.cache.Cache进行的wrapper，@since spring 3.2；spring4将此类更新到JCache 0.11版本；
 
另外，因为我们在应用中并不是使用一个Cache，而是多个，因此Spring还提供了CacheManager抽象，用于缓存的管理： 
Java代码  收藏代码
package org.springframework.cache;  
import java.util.Collection;  
public interface CacheManager {  
    Cache getCache(String name); //根据Cache名字获取Cache   
    Collection<String> getCacheNames(); //得到所有Cache的名字  
}  
 
默认提供的实现： 
ConcurrentMapCacheManager/ConcurrentMapCacheFactoryBean：管理ConcurrentMapCache；
GuavaCacheManager；
EhCacheCacheManager/EhCacheManagerFactoryBean；
JCacheCacheManager/JCacheManagerFactoryBean；
 
另外还提供了CompositeCacheManager用于组合CacheManager，即可以从多个CacheManager中轮询得到相应的Cache，如
Java代码  收藏代码
<bean id="cacheManager" class="org.springframework.cache.support.CompositeCacheManager">  
    <property name="cacheManagers">  
        <list>  
            <ref bean="ehcacheManager"/>  
            <ref bean="jcacheManager"/>  
        </list>  
    </property>  
    <property name="fallbackToNoOpCache" value="true"/>  
</bean>  
当我们调用cacheManager.getCache(cacheName) 时，会先从第一个cacheManager中查找有没有cacheName的cache，如果没有接着查找第二个，如果最后找不到，因为fallbackToNoOpCache=true，那么将返回一个NOP的Cache否则返回null。
 
 
除了GuavaCacheManager之外，其他Cache都支持Spring事务的，即如果事务回滚了，Cache的数据也会移除掉。
 
Spring不进行Cache的缓存策略的维护，这些都是由底层Cache自己实现，Spring只是提供了一个Wrapper，提供一套对外一致的API。
 
示例
需要添加Ehcache依赖，具体依赖轻参考pom.xml
 
SpringCacheTest.java
Java代码  收藏代码
@Test  
public void test() throws IOException {  
    //创建底层Cache  
    net.sf.ehcache.CacheManager ehcacheManager  
            = new net.sf.ehcache.CacheManager(new ClassPathResource("ehcache.xml").getInputStream());  
  
    //创建Spring的CacheManager  
    EhCacheCacheManager cacheCacheManager = new EhCacheCacheManager();  
    //设置底层的CacheManager  
    cacheCacheManager.setCacheManager(ehcacheManager);  
  
    Long id = 1L;  
    User user = new User(id, "zhang", "zhang@gmail.com");  
  
    //根据缓存名字获取Cache  
    Cache cache = cacheCacheManager.getCache("user");  
    //往缓存写数据  
    cache.put(id, user);  
    //从缓存读数据  
    Assert.assertNotNull(cache.get(id, User.class));  
}  
此处直接使用Spring提供的API进行操作；我们也可以通过xml/注解方式配置到spring容器；
 
xml风格的（spring-cache.xml）：
Java代码  收藏代码
<bean id="ehcacheManager" class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean">  
    <property name="configLocation" value="classpath:ehcache.xml"/>  
</bean>  
  
<bean id="cacheManager" class="org.springframework.cache.ehcache.EhCacheCacheManager">  
    <property name="cacheManager" ref="ehcacheManager"/>  
    <property name="transactionAware" value="true"/>  
</bean>  
spring提供EhCacheManagerFactoryBean来简化ehcache cacheManager的创建，这样注入configLocation，会自动根据路径从classpath下找，比编码方式简单多了，然后就可以从spring容器获取cacheManager进行操作了。此处的transactionAware表示是否事务环绕的，如果true，则如果事务回滚，缓存也回滚，默认false。
 
 
注解风格的（AppConfig.java）：
Java代码  收藏代码
@Bean  
public CacheManager cacheManager() {  
  
    try {  
        net.sf.ehcache.CacheManager ehcacheCacheManager  
                = new net.sf.ehcache.CacheManager(new ClassPathResource("ehcache.xml").getInputStream());  
  
        EhCacheCacheManager cacheCacheManager = new EhCacheCacheManager(ehcacheCacheManager);  
        return cacheCacheManager;  
    } catch (IOException e) {  
        throw new RuntimeException(e);  
    }  
}  
和编程方式差不多就不多介绍了。
另外，除了这些默认的Cache之外，我们可以写自己的Cache实现；而且即使不用之后的Spring Cache注解，我们也尽量使用Spring Cache API进行Cache的操作，如果要替换底层Cache也是非常方便的。到此基本的Cache API就介绍完了，接下来我们来看看使用Spring Cache注解来简化Cache的操作。
 
Cache注解
启用Cache注解
XML风格的（spring-cache.xml）：
Java代码  收藏代码
<cache:annotation-driven cache-manager="cacheManager" proxy-target-class="true"/>  
另外还可以指定一个 key-generator，即默认的key生成策略，后边讨论；
 
 
注解风格的（AppConfig.java）：
Java代码  收藏代码
@Configuration  
@ComponentScan(basePackages = "com.sishuok.spring.service")  
@EnableCaching(proxyTargetClass = true)  
public class AppConfig implements CachingConfigurer {  
    @Bean  
    @Override  
    public CacheManager cacheManager() {  
  
        try {  
            net.sf.ehcache.CacheManager ehcacheCacheManager  
                    = new net.sf.ehcache.CacheManager(new ClassPathResource("ehcache.xml").getInputStream());  
  
            EhCacheCacheManager cacheCacheManager = new EhCacheCacheManager(ehcacheCacheManager);  
            return cacheCacheManager;  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
  
    @Bean  
    @Override  
    public KeyGenerator keyGenerator() {  
        return new SimpleKeyGenerator();  
    }  
}  
1、使用@EnableCaching启用Cache注解支持；
2、实现CachingConfigurer，然后注入需要的cacheManager和keyGenerator；从spring4开始默认的keyGenerator是SimpleKeyGenerator；
 
@CachePut 

应用到写数据的方法上，如新增/修改方法，调用方法时会自动把相应的数据放入缓存： 
Java代码  收藏代码
@CachePut(value = "user", key = "#user.id")  
public User save(User user) {  
    users.add(user);  
    return user;  
}  
即调用该方法时，会把user.id作为key，返回值作为value放入缓存；
 
@CachePut注解：
Java代码  收藏代码
public @interface CachePut {  
    String[] value();              //缓存的名字，可以把数据写到多个缓存  
    String key() default "";       //缓存key，如果不指定将使用默认的KeyGenerator生成，后边介绍  
    String condition() default ""; //满足缓存条件的数据才会放入缓存，condition在调用方法之前和之后都会判断  
    String unless() default "";    //用于否决缓存更新的，不像condition，该表达只在方法执行之后判断，此时可以拿到返回值result进行判断了  
}  
 
@CacheEvict 
即应用到移除数据的方法上，如删除方法，调用方法时会从缓存中移除相应的数据：
Java代码  收藏代码
@CacheEvict(value = "user", key = "#user.id") //移除指定key的数据  
public User delete(User user) {  
    users.remove(user);  
    return user;  
}  
@CacheEvict(value = "user", allEntries = true) //移除所有数据  
public void deleteAll() {  
    users.clear();  
}  
 
@CacheEvict注解：
Java代码  收藏代码
public @interface CacheEvict {  
    String[] value();                        //请参考@CachePut  
    String key() default "";                 //请参考@CachePut  
    String condition() default "";           //请参考@CachePut  
    boolean allEntries() default false;      //是否移除所有数据  
    boolean beforeInvocation() default false;//是调用方法之前移除/还是调用之后移除  
 
@Cacheable
应用到读取数据的方法上，即可缓存的方法，如查找方法：先从缓存中读取，如果没有再调用方法获取数据，然后把数据添加到缓存中：
Java代码  收藏代码
@Cacheable(value = "user", key = "#id")  
 public User findById(final Long id) {  
     System.out.println("cache miss, invoke find by id, id:" + id);  
     for (User user : users) {  
         if (user.getId().equals(id)) {  
             return user;  
         }  
     }  
     return null;  
 }  
 
@Cacheable注解：
Java代码  收藏代码
public @interface Cacheable {  
    String[] value();             //请参考@CachePut  
    String key() default "";      //请参考@CachePut  
    String condition() default "";//请参考@CachePut  
    String unless() default "";   //请参考@CachePut    
 
运行流程
Java代码  收藏代码
1、首先执行@CacheEvict（如果beforeInvocation=true且condition 通过），如果allEntries=true，则清空所有  
2、接着收集@Cacheable（如果condition 通过，且key对应的数据不在缓存），放入cachePutRequests（也就是说如果cachePutRequests为空，则数据在缓存中）  
3、如果cachePutRequests为空且没有@CachePut操作，那么将查找@Cacheable的缓存，否则result=缓存数据（也就是说只要当没有cache put请求时才会查找缓存）  
4、如果没有找到缓存，那么调用实际的API，把结果放入result  
5、如果有@CachePut操作(如果condition 通过)，那么放入cachePutRequests  
6、执行cachePutRequests，将数据写入缓存（unless为空或者unless解析结果为false）；  
7、执行@CacheEvict（如果beforeInvocation=false 且 condition 通过），如果allEntries=true，则清空所有  
流程中需要注意的就是2/3/4步：
如果有@CachePut操作，即使有@Cacheable也不会从缓存中读取；问题很明显，如果要混合多个注解使用，不能组合使用@CachePut和@Cacheable；官方说应该避免这样使用（解释是如果带条件的注解相互排除的场景）；不过个人感觉还是不要考虑这个好，让用户来决定如何使用，否则一会介绍的场景不能满足。
提供的SpEL上下文数据

Spring Cache提供了一些供我们使用的SpEL上下文数据，下表直接摘自Spring官方文档：
名字	位置	描述	示例
methodName
root对象
当前被调用的方法名
#root.methodName
method
root对象
当前被调用的方法
#root.method.name
target
root对象
当前被调用的目标对象
#root.target
targetClass
root对象
当前被调用的目标对象类
#root.targetClass
args
root对象
当前被调用的方法的参数列表
#root.args[0]
caches
root对象
当前方法调用使用的缓存列表（如@Cacheable(value={"cache1", "cache2"})），则有两个cache
#root.caches[0].name
argument name
执行上下文
当前被调用的方法的参数，如findById(Long id)，我们可以通过#id拿到参数
#user.id
result
执行上下文
方法执行后的返回值（仅当方法执行之后的判断有效，如‘unless’，'cache evict'的beforeInvocation=false）
#result
通过这些数据我们可能实现比较复杂的缓存逻辑了，后边再来介绍。
 
Key生成器
如果在Cache注解上没有指定key的话@CachePut(value = "user")，会使用KeyGenerator进行生成一个key： 
Java代码  收藏代码
public interface KeyGenerator {  
    Object generate(Object target, Method method, Object... params);  
}  
默认提供了DefaultKeyGenerator生成器（Spring 4之后使用SimpleKeyGenerator）： 
Java代码  收藏代码
@Override  
public Object generate(Object target, Method method, Object... params) {  
    if (params.length == 0) {  
        return SimpleKey.EMPTY;  
    }  
    if (params.length == 1 && params[0] != null) {  
        return params[0];  
    }  
    return new SimpleKey(params);  
}  
即如果只有一个参数，就使用参数作为key，否则使用SimpleKey作为key。
 
我们也可以自定义自己的key生成器，然后通过xml风格的<cache:annotation-driven key-generator=""/>或注解风格的CachingConfigurer中指定keyGenerator。 
 

条件缓存
根据运行流程，如下@Cacheable将在执行方法之前( #result还拿不到返回值)判断condition，如果返回true，则查缓存； 
Java代码  收藏代码
@Cacheable(value = "user", key = "#id", condition = "#id lt 10")  
public User conditionFindById(final Long id)  
 
根据运行流程，如下@CachePut将在执行完方法后（#result就能拿到返回值了）判断condition，如果返回true，则放入缓存； 
Java代码  收藏代码
@CachePut(value = "user", key = "#id", condition = "#result.username ne 'zhang'")  
public User conditionSave(final User user)   
  
根据运行流程，如下@CachePut将在执行完方法后（#result就能拿到返回值了）判断unless，如果返回false，则放入缓存；（即跟condition相反）
Java代码  收藏代码
@CachePut(value = "user", key = "#user.id", unless = "#result.username eq 'zhang'")  
    public User conditionSave2(final User user)   
 
根据运行流程，如下@CacheEvict， beforeInvocation=false表示在方法执行之后调用（#result能拿到返回值了）；且判断condition，如果返回true，则移除缓存；
Java代码  收藏代码
@CacheEvict(value = "user", key = "#user.id", beforeInvocation = false, condition = "#result.username ne 'zhang'")  
public User conditionDelete(final User user)   
 
@Caching
有时候我们可能组合多个Cache注解使用；比如用户新增成功后，我们要添加id-->user；username--->user；email--->user的缓存；此时就需要@Caching组合多个注解标签了。
 
如用户新增成功后，添加id-->user；username--->user；email--->user到缓存； 
Java代码  收藏代码
@Caching(  
        put = {  
                @CachePut(value = "user", key = "#user.id"),  
                @CachePut(value = "user", key = "#user.username"),  
                @CachePut(value = "user", key = "#user.email")  
        }  
)  
public User save(User user) {  
  
@Caching定义如下： 

Java代码  收藏代码
public @interface Caching {  
    Cacheable[] cacheable() default {}; //声明多个@Cacheable  
    CachePut[] put() default {};        //声明多个@CachePut  
    CacheEvict[] evict() default {};    //声明多个@CacheEvict  
}  
 
自定义缓存注解
比如之前的那个@Caching组合，会让方法上的注解显得整个代码比较乱，此时可以使用自定义注解把这些注解组合到一个注解中，如： 
Java代码  收藏代码
@Caching(  
        put = {  
                @CachePut(value = "user", key = "#user.id"),  
                @CachePut(value = "user", key = "#user.username"),  
                @CachePut(value = "user", key = "#user.email")  
        }  
)  
@Target({ElementType.METHOD, ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Inherited  
public @interface UserSaveCache {  
}  
 
这样我们在方法上使用如下代码即可，整个代码显得比较干净。 
Java代码  收藏代码
@UserSaveCache  
public User save(User user)  
  
示例
新增/修改数据时往缓存中写 
Java代码  收藏代码
@Caching(  
        put = {  
                @CachePut(value = "user", key = "#user.id"),  
                @CachePut(value = "user", key = "#user.username"),  
                @CachePut(value = "user", key = "#user.email")  
        }  
)  
public User save(User user)  
 
Java代码  收藏代码
@Caching(  
        put = {  
                @CachePut(value = "user", key = "#user.id"),  
                @CachePut(value = "user", key = "#user.username"),  
                @CachePut(value = "user", key = "#user.email")  
        }  
)  
public User update(User user)   
删除数据时从缓存中移除
Java代码  收藏代码
@Caching(  
        evict = {  
                @CacheEvict(value = "user", key = "#user.id"),  
                @CacheEvict(value = "user", key = "#user.username"),  
                @CacheEvict(value = "user", key = "#user.email")  
        }  
)  
public User delete(User user)   
Java代码  收藏代码
@CacheEvict(value = "user", allEntries = true)  
 public void deleteAll()  
 
查找时从缓存中读
Java代码  收藏代码
@Caching(  
        cacheable = {  
                @Cacheable(value = "user", key = "#id")  
        }  
)  
public User findById(final Long id)   
 
Java代码  收藏代码
@Caching(  
         cacheable = {  
                 @Cacheable(value = "user", key = "#username")  
         }  
 )  
 public User findByUsername(final String username)   
 
Java代码  收藏代码
@Caching(  
          cacheable = {  
                  @Cacheable(value = "user", key = "#email")  
          }  
  )  
  public User findByEmail(final String email)  
  
 
问题及解决方案
一、比如findByUsername时，不应该只放username-->user，应该连同id--->user和email--->user一起放入；这样下次如果按照id查找直接从缓存中就命中了；这需要根据之前的运行流程改造CacheAspectSupport： 
Java代码  收藏代码
// We only attempt to get a cached result if there are no put requests  
if (cachePutRequests.isEmpty() && contexts.get(CachePutOperation.class).isEmpty()) {  
    result = findCachedResult(contexts.get(CacheableOperation.class));  
}  
改为：
Java代码  收藏代码
Collection<CacheOperationContext> cacheOperationContexts = contexts.get(CacheableOperation.class);  
if (!cacheOperationContexts.isEmpty()) {  
    result = findCachedResult(cacheOperationContexts);  
}  
然后就可以通过如下代码完成想要的功能： 
Java代码  收藏代码
@Caching(  
        cacheable = {  
                @Cacheable(value = "user", key = "#username")  
        },  
        put = {  
                @CachePut(value = "user", key = "#result.id", condition = "#result != null"),  
                @CachePut(value = "user", key = "#result.email", condition = "#result != null")  
        }  
)  
public User findByUsername(final String username) {  
    System.out.println("cache miss, invoke find by username, username:" + username);  
    for (User user : users) {  
        if (user.getUsername().equals(username)) {  
            return user;  
        }  
    }  
    return null;  
}  
  
 
二、缓存注解会让代码看上去比较乱；应该使用自定义注解把缓存注解提取出去；
 
三、往缓存放数据/移除数据是有条件的，而且条件可能很复杂，考虑使用SpEL表达式：
Java代码  收藏代码
@CacheEvict(value = "user", key = "#user.id", condition = "#root.target.canCache() and #root.caches[0].get(#user.id).get().username ne #user.username", beforeInvocation = true)  
public void conditionUpdate(User user)  
 或更复杂的直接调用目标对象的方法进行操作（如只有修改了某个数据才从缓存中清除，比如菜单数据的缓存，只有修改了关键数据时才清空菜单对应的权限数据） 
Java代码  收藏代码
@Caching(  
        evict = {  
                @CacheEvict(value = "user", key = "#user.id", condition = "#root.target.canEvict(#root.caches[0], #user.id, #user.username)", beforeInvocation = true)  
        }  
)  
public void conditionUpdate(User user)   
Java代码  收藏代码
public boolean canEvict(Cache userCache, Long id, String username) {  
    User cacheUser = userCache.get(id, User.class);  
    if (cacheUser == null) {  
        return false;  
    }  
    return !cacheUser.getUsername().equals(username);  
}  
如上方式唯一不太好的就是缓存条件判断方法也需要暴露出去；而且缓存代码和业务代码混合在一起，不优雅；因此把canEvict方法移到一个Helper静态类中就可以解决这个问题了：
Java代码  收藏代码
@CacheEvict(value = "user", key = "#user.id", condition = "T(com.sishuok.spring.service.UserCacheHelper).canEvict(#root.caches[0], #user.id, #user.username)", beforeInvocation = true)  
public void conditionUpdate(User user)  
 
四、其实对于：id--->user；username---->user；email--->user；更好的方式可能是：id--->user；username--->id；email--->id；保证user只存一份；如：
Java代码  收藏代码
@CachePut(value="cacheName", key="#user.username", cacheValue="#user.username")  
public void save(User user)   
Java代码  收藏代码
@Cacheable(value="cacheName", ley="#user.username", cacheValue="#caches[0].get(#caches[0].get(#username).get())")  
public User findByUsername(String username)  
 
五、使用Spring3.1注解 缓存 模糊匹配Evict的问题 
缓存都是key-value风格的，模糊匹配本来就不应该是Cache要做的；而是通过自己的缓存代码实现；
 
六、spring cache的缺陷：例如有一个缓存存放 list<User>，现在你执行了一个 update(user)的方法，你一定不希望清除整个缓存而想替换掉update的元素
这个在现有的抽象上没有很好的方案，可以考虑通过condition在之前的Helper方法中解决；当然不是很优雅。
 
也就是说Spring Cache注解还不是很完美，我认为可以这样设计：
@Cacheable(cacheName = "缓存名称",key="缓存key/SpEL", value="缓存值/SpEL/不填默认返回值",  beforeCondition="方法执行之前的条件/SpEL", afterCondition="方法执行后的条件/SpEL", afterCache="缓存之后执行的逻辑/SpEL")
value也是一个SpEL，这样可以定制要缓存的数据；afterCache定制自己的缓存成功后的其他逻辑。
 
 
当然Spring Cache注解对于大多数场景够用了，如果场景复杂还是考虑使用AOP吧；如果自己实现请考虑使用Spring Cache API进行缓存抽象。
 

