# Shiro原理

## 1、shiro登陆授权原理

1. shiro在创建用户的时候根据用户输入密码、生成随机salt值（存入数据库），加密算法和迭代次数生成加密的密码存储到数据库中

2. shiro进行登陆的时候，首先在自定义realm的doGetAuthenticationInfo方法中从前台的token判断用户是否存在，然后把所有的用户数据从数据库取出来，取出salt和经过加密的密码。shiro根据用户输入的密码经过salt和shiro配置号的相同的加密算法和迭代次数运算得到的密码和realm中取得的密码进行比对，一样则认证通过！（MD5盐值加密，数据库的salt生成盐值使用下面的方法）登陆成功把user信息存储到session中，前端在cookies中保存JSESSIONID，每次访问页面携带JSESSIONID用户判断用户会话状态（是否登陆过）。 Session的默认失效时间是30分钟，微信access_token的有效期是两小时。默认使用session判断。题外话：JWT(json web tokens)机制，登陆成功之后，服务器使用Jwt加密把token返回给客户端，然后客户端每次请求时候header携带token访问服务端，服务端设置过滤器拦截请求判断token是否合法、进行一系列处理。通过即可访问资源。JWT介绍https://blog.csdn.net/weixin_42109071/article/details/102509076 JWT算法介绍https://blog.csdn.net/u010288264/article/details/52004169

   ```
   //生成盐
   ByteSource credentialsSalt=ByteSource.Util.bytes(user.getSalt());
   ```

3. 数据库有多处可以配置多个realm，配置不同的加密方式。认证策略有三种：1、一个realm验证成功即可，返回第一个身份验证成功的信息 2、只要有一个realm验证成功即可，返回所有身份验证成功的信息 3、所有的realm验证成功才算成功，返回所有realm身份验证成功的信息 

   ModularRealmAuthenticator默认是AtLeastOneSuccessfulStrategy策略

   多realm授权只要有一个过了就行

4. Shiro的Session提供了完整的企业级会话管理的功能，不依赖于底层容器web容器tomcat，不管JavaEE和JavaSE环境都可以使用

5. SessionDao：a、AbstractSessionDAO提供了SessionDAO的基础实现如生成会话ID等。b、CachingSessionDAO提供了对开发者透明的会话缓存的功能，需要设置相应的CacheManager c、MemorySessionDAO直接在内存中进行会话维护 d、EnterpriseCacheSessionDAO提供了缓存功能的会话维护，默认情况下使用MapCache实现，内部使用ConcurrentHashMap保存缓存的会话

6. Shiro缓存：CacheManagerAware接口，Shiro内部相应的组件会自动检测相应的对象如Realm是否实现了CacheManagerAware并自动注入相应的CacheManager，Realm是有缓存的。Shiro提供了记住我RememberMe功能，subject.isAuthenticated()表示用户进行身份验证登陆的，subject.isRemember()表示用户是通过记住我登陆的，二者只能选一个。可以设置cookies的maxAge时长来修改RememberMe时长







