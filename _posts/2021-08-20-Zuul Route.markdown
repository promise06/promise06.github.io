---
layout: post
title:  "Zuul Route 流程"
date:   2021-08-20 16:07:06 +0800
categories: netflix zuul ribbon hystrix
---

SpringCloud Netflix Core 版本：1.3.x

最近在开发网关，了解一下SpringCloud 封装之后的Zuul的工作流程。注意一下版本号，Zuul 1和2 之间差别还是很大的，Netflix官网对于Zuul 1和2版本的工作原理都有详细的解释，可以参见 https://netflixtechblog.com/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee。本文从代码的层面去解释一下Zuul 1 在被作为反向代理时的工作原理,注意：所有的配置参数均使用默认值。

前置知识： [RxJava][Reactivex.io]

我把Zuul处理请求简单的分成了3个阶段：
第一阶段，请求是怎么到达ZuulServlet的
第二阶段，请求对应的目标服务是怎么被定为到的
第三阶段，请求是怎么到达目标服务的
下面我们一个阶段一个阶段的解释。



第一个阶段：当需要创建一个Zuul 的反向代理时，我们会用到@EnableZuulProxy 这个注解

```java

@EnableCircuitBreaker
@EnableDiscoveryClient
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(ZuulProxyMarkerConfiguration.class)
public @interface EnableZuulProxy {
}

```
在这个注解中需要注意@EnableCircuitBreaker，这个注解指明Zuul反向代理默认启用断路器，@EnableDiscoveryClient这个注解指明默认使用服务发现，@Import(ZuulProxyMarkerConfiguration.class)这个注解是一个配置类的标识，具体的代码如下：

```java

@Configuration
public class ZuulServerMarkerConfiguration {
	@Bean
	public Marker zuulServerMarkerBean() {
		return new Marker();
	}

	class Marker {
	}
}

```
一旦这个配置类存在，那么就会使用 ZuulProxyAutoConfiguration 作为Zuul的配置类，这个类继承了 ZuulServerAutoConfiguration并且丰富了配置，增加了新的过滤器,RouteLocator和监听器，如下：
```java
	@Bean
	@ConditionalOnMissingBean(DiscoveryClientRouteLocator.class)
	public DiscoveryClientRouteLocator discoveryRouteLocator() {
		return new DiscoveryClientRouteLocator(this.server.getServletPrefix(), this.discovery, this.zuulProperties,
				this.serviceRouteMapper);
	}

	// pre filters
	@Bean
	public PreDecorationFilter preDecorationFilter(RouteLocator routeLocator, ProxyRequestHelper proxyRequestHelper) {
		return new PreDecorationFilter(routeLocator, this.server.getServletPrefix(), this.zuulProperties,
				proxyRequestHelper);
	}

	// route filters
	@Bean
	public RibbonRoutingFilter ribbonRoutingFilter(ProxyRequestHelper helper,
			RibbonCommandFactory<?> ribbonCommandFactory) {
		RibbonRoutingFilter filter = new RibbonRoutingFilter(helper, ribbonCommandFactory, this.requestCustomizers);
		return filter;
	}

	@Bean
	@ConditionalOnMissingBean(SimpleHostRoutingFilter.class)
	public SimpleHostRoutingFilter simpleHostRoutingFilter(ProxyRequestHelper helper, ZuulProperties zuulProperties) {
		return new SimpleHostRoutingFilter(helper, zuulProperties);
	}

	@Bean
	public ApplicationListener<ApplicationEvent> zuulDiscoveryRefreshRoutesListener() {
		return new ZuulDiscoveryRefreshListener();
	}

```











[Reactivex.io] : http://reactivex.io/intro.html