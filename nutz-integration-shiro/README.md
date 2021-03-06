Nutz集成Shiro的插件
======================

简介(可用性:生产)
==================================

集成Shiro的登陆,鉴权,和Session机制

本插件的主要部件
-------------------------

* SimpleAuthenticationFilter 穿透登陆请求到Nutz.Mvc的入口方法
* SimpleShiroToken 不带校验信息的token实现
* ShiroSessionProvider 在Nutz.MVC作用域内,使用Shiro替换容器原生的Session机制
* ShiroProxy 用于模板引擎中方便调用shiro

**原CaptchaFormAuthenticationFilter已经废弃** 原因是出错了都不知道哪里错,而且不好定制.


具体实例,请参考[NutzCN论坛的源码](https://github.com/wendal/nutz-book-project)

使用方法
-------------------------

* 添加本插件及shiro的依赖, 支持1.2+版本,建议用最新版
* 继承AbstractRealm,实现一个Realm. 特别注意在构造方法内注册关联的Token类!!
* 添加 shiro.ini
* 在web.xml中添加ShiroFilter
* 添加入口方法完成登陆
* 可选: 注册ShiroSessionProvider
* 可选: 登记UU32SessionIdGenerator

添加本插件及依赖
-----------------------------

```xml
		<dependency>
			<groupId>org.nutz</groupId>
			<artifactId>nutz-integration-shiro</artifactId>
			<version>1.r.56</version>
		</dependency>
		<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-core</artifactId>
			<version>1.3.0</version>
		</dependency>
		<!-- 下面的是缓存相关的,可选 -->
		<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-all</artifactId>
			<version>1.3.0</version>
			<exclusions>
				<exclusion>
					<artifactId>ehcache-core</artifactId>
					<groupId>net.sf.ehcache</groupId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>net.sf.ehcache</groupId>
			<artifactId>ehcache</artifactId>
			<version>2.10.1</version>
		</dependency>
```

继承AbstractRealm,实现一个Realm
--------------------------------------

这部分是跟具体项目的Pojo类紧密结合的,所以没有给出默认实现.

请参考[NutzCN论坛的源码](https://github.com/wendal/nutz-book-project)中的net.wendal.nutzbook.shiro.realm.SimpleAuthorizingRealm类

**请特别注意构造方法中注册的Token类** 在入口方法中执行SecurityUtils.getSubject().login时所使用的Token类型必须匹配!!

最简单的shiro.ini
--------------------------

其中的net.wendal.nutzbook.shiro.realm.NutDaoRealm是nutzbook中的NutDaoRealm实现.

	[main]
	nutzdao_realm = net.wendal.nutzbook.shiro.realm.NutDaoRealm
	authc = org.nutz.integration.shiro.SimpleAuthenticationFilter
	authc.loginUrl  = /user/login

	[urls]
	/user/logout = logout
	
web.xml中添加ShiroFilter配置
----------------------------

必须添加在NutFilter之前,让其先于NutFilter进行初始化

```xml
	<listener>
		<listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
	</listener>
	<filter>
		<filter-name>ShiroFilter</filter-name>
 		<filter-class>org.apache.shiro.web.servlet.ShiroFilter</filter-class>
	</filter>
	<filter-mapping>
		<filter-name>ShiroFilter</filter-name>
		<url-pattern>/*</url-pattern>
		<dispatcher>REQUEST</dispatcher>
		<dispatcher>FORWARD</dispatcher>
		<dispatcher>INCLUDE</dispatcher>
		<dispatcher>ERROR</dispatcher>
	</filter-mapping>
```

添加登陆用的入口方法
--------------------------

```java
    // 映射 /user/login , 与shiro.ini对应.
	@POST
	@Ok("json")
	@At
	public Object login(@Param("username")String username, 
					  @Param("password")String password,
					  @Param("rememberMe")boolean rememberMe,
					  @Param("captcha")String captcha) {
		NutMap re = new NutMap().setv("ok", false);
		// 如果已经登陆过,直接返回真
		Subject subject = SecurityUtils.getSubject();
		if (subject.isAuthenticated())
		    return re.setv("ok", true);
		// 检查用户名密码验证码是否正确,这里就不写出来了.
		// ............
		// 检查用户名密码
		User user = dao.fetch(User.class, username);
		if (user == null) {
			return re.setv("msg", "用户不存在");
		}
		// 比对密码, 严重建议用hash和加盐!!
		String face = new Sha256Hash(password, user.getSalt()).toHex();
		if (!face.equalsIgnoreCase(user.getPassword())) {
			return re.setv("msg", "密码错误");
		}
		subject.login(new SimpleShiroToken(user.getId()));
		// 需要放点东西进session,如果配置了ShiroSessionProvider,下面两种代码等价
		// req.getSession().setAttribute("me", user);
		// subject.getSession().setAttribute("me", user);
		return re.setv("ok", true);
	}
```

ShiroSessionProvider用法
--------------------------

在MainModule中的配置

	@SessionBy(ShiroSessionProvider.class)
	
这个功能是可选,也是推荐的,配合ehcache/redis,可以实现session持久化

配置该SessionProvider后, nutz.mvc作用域内的req.getHttpSession均返回shiro的Session.

UU32SessionIdGenerator 用法
---------------------------

在shiro.ini内添加:

    # use R.UU32()
    sessionIdGenerator = org.nutz.integration.shiro.UU32SessionIdGenerator
    securityManager.sessionManager.sessionDAO.sessionIdGenerator = $sessionIdGenerator
	
常见问题
---------------------------

**使用CaptchaFormAuthenticationFilter出现各种问题无法解决的话,请立即更换为SimpleAuthenticationFilter**

1. 使用CaptchaFormAuthenticationFilter, 账号密码均正确,但依然无法登陆 -- 必须带验证码,请查看其executeLogin方法
2. 使用CaptchaFormAuthenticationFilter,登陆就404,但事实上已经登陆 -- 若已经登陆,那么再次登陆时穿透的,如果后端没有入口方法对应,就会404.

3. 使用SimpleAuthenticationFilter, 就XXX -- 还没人遇到过问题,因为登陆操作在入口方法内,由你控制!!