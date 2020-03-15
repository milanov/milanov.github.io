---
layout: post
title:  "Routing Options In Spring MVC"
date:   2015-10-13 17:30:00
tags: spring spring-mvc routing reverse-routing
description: Spring MVC routing and reverse-routing options, urls generation
image: /assets/article_images/2015-10-13-routing-options-in-spring-mvc/rails-routes.png
---

Unlike many other popular web frameworks ([Rails](http://rubyonrails.org/){:rel='nofollow' target='_blank'}, [Grails](https://grails.org/){:rel='nofollow' target='_blank'}, [Django](https://www.djangoproject.com/){:rel='nofollow' target='_blank'}, [Play Framework](https://www.playframework.com/){:rel='nofollow' target='_blank'}, [Laravel](http://laravel.com){:rel='nofollow' target='_blank'}), Spring MVC's url mappings are dispersed throughout the whole application and not externalized in a single place. While there are good and bad sides to both approaches, this post will discuss the available options for routing management in Spring MVC and especially how application URL's are generated.

### 0) Out-of-the-box routing
Spring MVC 4 [handles requests mapping](http://static.springsource.org/spring/docs/4.0.x/spring-framework-reference/html/mvc.html){:rel='nofollow' target='_blank'} with `RequestMappingHandlerMapping` and `RequestMappingHandlerAdapter` beans (that's the "out-of-the-box" configuration that comes with a springmvc application). Url's are mapped to controllers and actions using the `@RequestMapping` annotation.
{% highlight java %}
@Controller
@RequestMapping("/hello")
public class HelloController {

    @RequestMapping("/{word}", method = GET)
    public @ResponseBody String hello(@PathVariable String word) {
        return "Hello " + word;
    }
}
{% endhighlight %}

To then construct a URL to a given resource or action, either some string manipulation or using a string-formatting library is is needed:
{% highlight java %}
String word = "world";
new URI("hello/" + word)                     => "/hello/world"
new UriTemplate("hello/{word}").expand(word) => "/hello/world"
{% endhighlight %}

While this method is simple, easy-to-use and works OOTB, it has its drawbacks - it requires fiddling with strings and knowing the mapping of actions by heart to construct an URL. The replication of mappings everywhere leads to a very difficult refactoring process, and there is no way to see all routes in one place. 

### 1) Externalizing routes in constants
An approach suggested by some people (and which I have used in commercial projects) is to still use `@RequestMapping`'s, but externalize the actual string routes in a constants file. For the `HelloController` example above that could look like this:

**Routes.java**
{% highlight java %}
String HELLO_PATH = "/hello";
String HELLO_WORD_PATH = "/{word}";
String HELLO_WORD_PATH_FULL = HELLO_PATH
	+ HELLO_WORD_PATH; // => "/hello/{word}"
{% endhighlight %}

This changes the request mappings to `@RequestMapping(Routes.HELLO_PATH)` and `@RequestMapping(Routes.HELLO_WORD_PATH)` respectively. To construct URL's a library or a custom helper method can be used:
{% highlight java %}
new UriTemplate(Routes.HELLO_WORD_PATH_FULL).expand(word);
{% endhighlight %}

This method improves over the default one, since at least we're not duplicating routes all over the place. It could be argued that we can even see all routes in one place (although nothing guarantees that). On the other hand it also has its cons - it may still be error-prone if the wrong constant is used (nothing prevents us from creating a link to `HELLO_WORD_PATH_FULL` when in fact we want to link to the `UsersController` for example). Other minor concerns include the fact that paths are no longer visible at first sight in the controller and the "conceptual" issue that that is not so much linking but merely a string replacement. 

### 2) Utilizing Spring MVC's reverse routing
From version `4.1.0` on Spring includes a reverse routing functionality. The method is called [fromMappingName](https://github.com/spring-projects/spring-framework/blob/6b129c52e3bbcc4d301ad7604d43f39c64d346a1/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/MvcUriComponentsBuilder.java#L241){:rel='nofollow' target='_blank'}, inside `MvcUriComponentsBuilder`. An example of its usage can be found in its Javadoc, but it basically boils down to calling it with a string formed by the initials of the controller, a number sign and the name of an action. For the `HelloController` that would mean:
{% highlight java %}
fromMappingName("HC#hello").arg(0, "world").build()
=> "/hello/world"
{% endhighlight %}

The fact that it returns the relative path to an action doesn't hint it, but internally the method requires a Spring context. That is potentially a drawback of this method - it can't be used outside of a Spring environment, like tests. The other drawback that it involves passing strings whoose refactoring may not be the easiest task. Also, some ambiguities can arise (`PersonController#create` and `PostController#create`, but they can be mitigated by providing custom names for the mappings through the `name` attribute of a `@RequestMapping`.

### 3) Utilizing Spring HATEOAS
The HATEOAS project from Spring provides many features, but among them is a very useful type-safe linking-to-controller-actions method:
{% highlight java %}
linkTo(methodOn(HelloController.class).hello(word).toUri()
=> "http://host/root-path/hello/world"
{% endhighlight %}

The returned URI is absolute (`http://host/root-path/hello/world`), since it uses some of the same classes as the previously described reverse routing. This again means that using it in tests is not an options, but at least it doesn't involve passing strings and is logically more precise. The only other small drawback that could be found is that linking to actions which accept other parameters than `@PathParam`'s need `null`'s as arguments (for example `UsersController#list(Pageable pageable)` would be called like `linkTo(methodOn(UsersController.class).list(null).toUri()`.

The need for relative URLs and to not using a context is asked for in [this github issue](https://github.com/spring-projects/spring-hateoas/issues/330){:rel='nofollow' target='_blank'}. Since it's still open, I came up with the following helper method (which uses a lot from Spring HATEOAS):

{% highlight java %}
private static final MappingDiscoverer DISCOVERER = new AnnotationMappingDiscoverer(RequestMapping.class);

public static URI linkTo(Object invocationValue) {
    LastInvocationAware invocations = (LastInvocationAware) invocationValue;
    DummyInvocationUtils.MethodInvocation invocation = invocations.getLastInvocation();

    String mapping = DISCOVERER.getMapping(invocation.getTargetType(), invocation.getMethod());

    return new UriTemplate(mapping).expand(invocation.getArguments());
}
{% endhighlight %}

It is probably not perfect, but for now it works and covers all use-cases that I've stumbled upon. The usage is the same as for the original `linkTo` method, only without the final `toUri()` call.
{% highlight java %}
linkTo(methodOn(HelloController.class).hello(word) => "/hello/world"
{% endhighlight %}

### Bonus) Using externalized routing 
Externalized request mapping is [on its way](https://jira.spring.io/browse/SPR-5757){:rel='nofollow' target='_blank'} in version `5.0` of Spring MVC. Until then a library called \*surprise surprise\* [springmvc-router](http://resthub.org/springmvc-router/){:rel='nofollow' target='_blank'} is available to fill-in the gap. It provides simple mapping between actions and URLs:
{% highlight java %}
    GET     /user/?                 userController.listAll
    GET     /user/{<[0-9]+>id}      userController.showUser
{% endhighlight %}
Obtaining the URL to an action is done by reverse routing, e.g. `Router.reverse("userController.listAll")`. This shares the same drawbacks like the currently available reverse-routing, but at least probably makes refactoring easier and handles URL priority.