---
slug: spring-trick-1
title: Spring Trick - 重载方法参数继承注解
authors: higan
tags: [ Spring, Java, Kotlin ]
---

在基于 Spring 的后端技术栈中，经常会将 Feign Client 与 Spring MVC 结合使用，Feign Client 负责维持 API 接口约定，Spring MVC
的 Controller 则负责实现 API。当一个 Controller 实现 Feign Client 接口时，方法上的注解可以不用再写，但是参数的注解则需要再写一遍，本文会介绍通过歪门邪道的方式，让
Spring 支持读取重载方法的参数继承的注解。
<!--truncate-->
# 关于 Spring Trick

本系列将主要介绍一些关于 Spring
的歪门邪道用法与技巧，我也不知道会有几期，可能也就这么一期吧。由于都是些歪门邪道的做法，我这里为了实现效果，可能会不择手段，如果不是十分了解其中的工作原理，请绝对不要模仿！另外，本文所有代码皆为
Kotlin，珍爱生命，远离 Java。

# Feign Client 与 Controller

如果一个后端程序是基于 JVM 的，那么基本上 90% 都是基于 Spring 了，这个时候为了多个服务的 API 接口之间调用方便，一般会把项目分为两个工程。一个是
Feign Client，只是定义一些 Feign Client，从而定义整个 API 的接口与调用约定。另一个则是真正的 Spring 项目，这里会从之前的
Feign Client 接口创建 Controller，从而实现接口实现与调用的约定。

大概就是这样的感觉：

```kotlin
// Feign Client 定义
@FeignClient(name = "Api")
interface ApiFeignClient {
    @RequestMapping(value = "/api/test", method = arrayOf(RequestMethod.GET))
    fun getTestResult(): ResultModel
}

// Controller 实现，通过 Feign Client 保证调用约定
@RestController
class ApiController : ApiFeignClient {
    override fun getTestResult(): ResultModel {
        // Business code here
    }
}
```

但是如果当一个 API 接口具有多种类型的参数，这种看上去非常优雅的写法，可能就变得不那么优雅了。

```kotlin
@FeignClient(name = "Editor")
interface ApiFeignClient {
    @RequestMapping(value = "/api/test/{id}", method = arrayOf(RequestMethod.GET))
    fun getTestResult(@PathVariable("id") id: String, @RequestParam(value = "paging", required = false) paging:String?): ResultModel;
}

@RestController
public class ApiController implements ApiFeignClient {
    @Override
     fun getTestResult(@PathVariable("id") id: String, @RequestParam(value = "paging", required = false) paging:String?): ResultModel{
         // Business code here
    }
}
```

虽然方法上面的`RequestMapping`注解不需要重新写一遍，但是像参数上面的注解还是需要再重新写一遍，而且需要保证两边一致，这样让
Feign Client 在接口调用约定上少了一截作用。还是有一些调用约定是基于代码上一致，在开发过程中非常容易遗漏这一点，导致一些 BUG
的出现。

# 核心原理

为什么接口参数上的注解，在实现其接口的类上就没有了呢？这是由于编译时重写方法相当于创建了一个全新的方法，两个方法不是同一个
Method 反射对象，自然也就是不能获取到了。  
Java 或者 Kotlin 的本身的反射是无法直接通过类方法获取到其接口方法上的参数注解的，所以这里我们需要为其写一些工具方法和拓展方法。

```kotlin
object ClassUtils {
    fun getInheritedParamAnnotations(method: Method, index: Int): List<Annotation> {
        val type = method.declaringClass
        return getAnnotationsComplete(type, method, index)
    }

    inline fun <reified T> getInheritedParamAnnotation(method: Method, index: Int): T?{
        return getAnnotationsComplete(method, index).firstOrNull{ it is T } as T?
    }

    private fun getInheritedParamAnnotations(clazz: Class<*>?, method: Method, index: Int): List<Annotation> {
        clazz ?: return emptyList()
        if (index > method.parameterAnnotations.size) {
            throw IllegalArgumentException("Index out of bound")
        }

        val set = try {
            clazz.getDeclaredMethod(method.name, *method.parameterTypes).let {
                it.parameterAnnotations[index].toMutableSet()
            }
        }catch (exception: NoSuchMethodException){
            hashSetOf<Annotation>()
        }

        set += clazz.interfaces.map { getAnnotationsComplete(it, method, index) }.flatten()
        set += getAnnotationsComplete(clazz.superclass, method, index)
        return set.toList()
    }
}
```

原理很简单，大概分为这么几步：

01. 先从当前方法获取到方法所在的类，通过反射先获取到这个方法参数的注解
02. 然后获取到当前这个类实现接口，通过反射尝试获取签名相同的方法，然后获取参数的注解
03. 最后再拿到基类，一直递归下去即可
04. 最后的最后，所有的结果去重，就是我们要的注解了

这个的排序是当前类方法最优先，其次是实现的接口方法，然后是父类方法

# 不修改 Spring 代码实现

上面介绍的方法需要修改 Spring 的代码，在现在官方没有实现的情况下，自己编译一整套 Spring
是很麻烦的事情，而且还有各种依赖管理，所以接下来就开始我们的歪门邪道了。

## 通过继承 Spring 本身的 MethodArgumentResolver 实现

经过查看 Spring 的源码，简单的了解 Spring 的`PathVariable`注解的实现与原理，其实就是有一个`MethodArgumentResolver`
会在请求到来的时候，来将 HTTP 请求上的各种参数解释成参数，并传给 Controller，这个`MethodArgumentResolver`
会读取注解来判断应该从什么地方拿参数，例如 Query String，Request Body 与 请求 URL 的占位符之类的。

这里我们先从最简单的`PathVariableMethodArgumentResolver`开始，这个类就是用于解析 PathVariable 注解并提供参数给
Controller。

```kotlin
class SuperClassPathVariableMethodArgumentResolver: PathVariableMethodArgumentResolver() {
    override fun supportsParameter(parameter: MethodParameter): Boolean {
        val superResult = super.supportsParameter(parameter)
        if(superResult){
            return false
        }

        return parameter.getInheritedParamAnnotation<PathVariable>() != null
    }

    override fun createNamedValueInfo(parameter: MethodParameter): AbstractNamedValueMethodArgumentResolver.NamedValueInfo {
        val annotation = parameter.getInheritedParamAnnotation<PathVariable>() ?: throw Exception()
        return PathVariableNamedValueInfo(annotation)
    }

    override fun contributeMethodArgument(parameter: MethodParameter, value: Any?,
                                          builder: UriComponentsBuilder, uriVariables: MutableMap<String, Any>, conversionService: ConversionService) {
        val annotation = parameter.getInheritedParamAnnotation<PathVariable>()
        val name = annotation?.name ?: parameter.parameterName
        uriVariables.put(name, formatUriValue(conversionService, TypeDescriptor(parameter.nestedIfOptional()), value))
    }

    private class PathVariableNamedValueInfo(annotation: PathVariable) : AbstractNamedValueMethodArgumentResolver.NamedValueInfo(annotation.name, annotation.required, ValueConstants.DEFAULT_NONE)
}
```

通过继承`PathVariableMethodArgumentResolver`并重载了几个关键方法，将原来的直接从参数上面拿注解改成了通过之前我们写的能获取继承的注解的方法，就可以了。

对于`RequestParamMethodArgumentResolver`也是如法炮制。

```kotlin
class SuperClassRequestParamMethodArgumentResolver: RequestParamMethodArgumentResolver {
    private val useDefaultResolution: Boolean

    constructor(useDefaultResolution: Boolean): super(useDefaultResolution) {
        this.useDefaultResolution = useDefaultResolution
    }

    constructor(beanFactory: ConfigurableBeanFactory, useDefaultResolution: Boolean): super(beanFactory, useDefaultResolution) {
        this.useDefaultResolution = useDefaultResolution
    }

    override fun supportsParameter(parameter: MethodParameter): Boolean {
        val superResult = super.supportsParameter(parameter)
        if(superResult){
            return false
        }

        return parameter.getInheritedParamAnnotation<RequestParam>() != null
    }

    override fun createNamedValueInfo(parameter: MethodParameter): AbstractNamedValueMethodArgumentResolver.NamedValueInfo {
        val annotation = parameter.getInheritedParamAnnotation<RequestParam>()
        return annotation?.let { RequestParamNamedValueInfo(it) } ?: RequestParamNamedValueInfo()
    }

    override fun contributeMethodArgument(parameter: MethodParameter, value: Any?,
                                          builder: UriComponentsBuilder, uriVariables: Map<String, Any>, conversionService: ConversionService) {
        val annotation = parameter.getInheritedParamAnnotation<RequestParam>()
        val name = annotation?.name ?: parameter.parameterName

        when (value) {
            null -> {
                annotation?.let {
                    if (!it.required || it.defaultValue != ValueConstants.DEFAULT_NONE) {
                        return
                    }
                }
                builder.queryParam(name)
            }
            is Collection<*> -> for (element in (value as Collection<*>?)!!) {
                builder.queryParam(name, formatUriValue(conversionService, TypeDescriptor.nested(parameter, 1), element))
            }
            else -> builder.queryParam(name, formatUriValue(conversionService, TypeDescriptor(parameter), value))
        }
    }

    private class RequestParamNamedValueInfo : AbstractNamedValueMethodArgumentResolver.NamedValueInfo {
        constructor() : super("", false, ValueConstants.DEFAULT_NONE)

        constructor(annotation: RequestParam) : super(annotation.name, annotation.required, annotation.defaultValue)
    }
}
```

为了简单起见，我在这里做了些许功能的简化，例如不支持 Map 的 Request Param 之类的。

最后是`RequestResponseBodyMethodProcessor`这个东西有些复杂，我们放到写一个章节再说。

## Bean 的循环依赖

`RequestResponseBodyMethodProcessor` 的构造函数是需要有 Message Converter 与 Request Response Body Advice
的，这些东西我们在构造时基本上是不可能有的。
由于 Spring 的注入机制问题，我们自己写的`MethodArgumentResolver`通过`WebMvcConfigurerAdapter`注入时，总是会在 Spring
默认的`MethodArgumentResolver`前面。
这个时候 Message Converter 与 Request Response Body Advice
基本上也没有创建好，这个东西又存放于一个类型为`RequestMappingHandlerAdapter`的 Bean 里面。
那么，我们就遇到了 Bean 的循环依赖的问题，要想加入我们自己定义的`MethodArgumentResolver`
必须要先有`RequestMappingHandlerAdapter`，而`RequestMappingHandlerAdapter`是在我们自定义的配置创建完成之后才会注入。
`RequestMappingHandlerAdapter`等着我们完成注入，我们需要等`RequestMappingHandlerAdapter`
完成注入，直接在`WebMvcConfigurerAdapter`中加上自动填装的`RequestMappingHandlerAdapter`就会导致整个 Spring 应用根本跑起不来。
那么，如何解决这个问题呢？要想完成整个注入过程，那么势必需要有一方让步，`RequestMappingHandlerAdapter`
是不可能让步的，那就只能我们让步了。我这里采取了惰性构建，与套壳注入的原理，让我们的`MethodArgumentResolver`能够稍晚一步创建。

```kotlin
class SuperClassRequestBodyMethodArgumentResolver(val context: ApplicationContext) : HandlerMethodArgumentResolver {
    var requestBodyMethodProcessor: SuperClassRequestBodyMethodProcessor? = null

    override fun resolveArgument(parameter: MethodParameter, mavContainer: ModelAndViewContainer, webRequest: NativeWebRequest, binderFactory: WebDataBinderFactory): Any {
        createProcessor()
        return requestBodyMethodProcessor?.resolveArgument(parameter, mavContainer, webRequest, binderFactory) ?: throw Exception("Create 'SuperClassRequestBodyMethodProcessor' failed")
    }

    override fun supportsParameter(parameter: MethodParameter): Boolean {
        createProcessor()
        return requestBodyMethodProcessor?.supportsParameter(parameter) ?: throw Exception("Create 'SuperClassRequestBodyMethodProcessor' failed")

    }

    private fun createProcessor(){
        if(requestBodyMethodProcessor == null){
            val adapter = context.getBean(RequestMappingHandlerAdapter::class.java)
            requestBodyMethodProcessor = SuperClassRequestBodyMethodProcessor(adapter.messageConverters, adapter.getPrivateField<List<Any>>("requestResponseBodyAdvice") ?: throw Exception("Can't get 'requestResponseBodyAdvice'"))
        }
    }

    class SuperClassRequestBodyMethodProcessor : RequestResponseBodyMethodProcessor {
        constructor(converters: List<HttpMessageConverter<*>>) : super(converters) {
        }

        constructor(converters: List<HttpMessageConverter<*>>,
                    manager: ContentNegotiationManager) : super(converters, manager) {
        }

        constructor(converters: List<HttpMessageConverter<*>>,
                    requestResponseBodyAdvice: List<Any>) : super(converters, null, requestResponseBodyAdvice) {
        }

        constructor(converters: List<HttpMessageConverter<*>>,
                    manager: ContentNegotiationManager, requestResponseBodyAdvice: List<Any>) : super(converters, manager, requestResponseBodyAdvice) {
        }

        override fun supportsParameter(parameter: MethodParameter): Boolean {
            return parameter.getInheritedParamAnnotation<RequestBody>() != null
        }

        override fun checkRequired(parameter: MethodParameter): Boolean {
            return (parameter.getInheritedParamAnnotation<RequestBody>()?.required ?: false) && !parameter.isOptional

        }
    }
}
```

在上述代码中，有一个`SuperClassRequestBodyMethodArgumentResolver`类，它只是一个壳，让 Spring
在早期的时候注入进去，其实它什么也不会干，它的构造参数有一个当前应用的上下文，通过这个上下文，我们在之后可以拿到注入好的`RequestMappingHandlerAdapter`。
`SuperClassRequestBodyMethodProcessor`
类，才是真正干活的类，但是他的构造参数所需要的条件在注入的时候没有，所以我们在这个时候通过其套壳的类`SuperClassRequestBodyMethodArgumentResolver`
，先把实例注入进去，在第一次调用，也就是第一个请求到来的时候，再构建`SuperClassRequestBodyMethodProcessor`
这个时候肯定能够通过应用上下文获取到`RequestMappingHandlerAdapter`。完美的回避了循环依赖的问题。

## 私有字段？！

`RequestMappingHandlerAdapter` 中可以直接获取到 Message Converter，但是 Request Response Body Advice 却只有
setter，无法直接获取到，这个时候就需要祭出大杀器--反射了。
通过反射，我们可以直接获取到一个对象的私有字段的值，虽然，并不推荐这么做。反正我们这次是歪门邪道用法，也没办法了。

```kotlin
object ClassUtils {
    fun <T> getPrivateField(any: Any, name: String): T?{
        return any.javaClass.getDeclaredField(name).apply { isAccessible = true }.get(any) as T?
    }
}
```

直接拿到反射出来的 Field 对象，然后将其可访问性改成 true，这样所就能通过这个获取私有字段的值。

实现了这些之后我们就可以获得一个十分清爽的 Controller 了，再也不用担心忘记加注解了。

```kotlin
@FeignClient(name = "Editor")
interface ApiFeignClient {
    @RequestMapping(value = "/api/test/{id}", method = arrayOf(RequestMethod.GET))
    fun getTestResult(@PathVariable("id") id: String, @RequestParam(value = "paging", required = false) paging:String?): ResultModel;
}

@RestController
public class ApiController implements ApiFeignClient {
    @Override
     fun getTestResult(id: String, paging:String?): ResultModel{
         // Business code here
    }
}
```

# 修改 Spring 代码的实现方案

其实终极的解决方案应该是让 Spring 本身支持获取继承参数注解，本文就不详细说明了，Java 版的代码与如何修改 Spring
的代码实现该功能，可以参考这个 [SPR-16216](https://github.com/spring-projects/spring-framework/pull/1600/files)