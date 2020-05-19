In this post I am going to talk about how to implement ** filters ** in ** Spring **. Filters are those that can be set when an HTTP request is received. That is, assuming that we have a program listening in some URIs, we can specify that we want to execute something before the requests are processed by the controller.
This is very useful if we want all requests to meet a requirement, for example include a specific header.
To understand how filters work in ** Spring ** I have made a program that I will explain little by little.
You have the source code of the program on [my GITHUB page] (https://github.com/chuchip/SpringFilter)
I'll start by showing the controller for REST requests that is in the `PrincipalController.java` class. This will be in charge of managing all requests.

java
@RestController
public class PrincipalController {
@Autowired
SillyLog sillyLog;

@GetMapping ("*")
public String entryOther (HttpServletRequest request, HttpServletResponse response)
{
sillyLog.debug ("In entryOther");
if (response.getHeader ("PROFE")! = null)
sillyLog.debug ("Header contains PROFE:" + response.getHeader ("PROFE"));
if (response.getHeader ("CAKE")! = null)
sillyLog.debug ("Header contains CAKE:" + response.getHeader ("CAKE"));
return "returning by function entryOther \ r \ n" +
sillyLog.getMessage ();
}
@GetMapping (value = {"/", "one"})
public String entryOne (HttpServletRequest request, HttpServletResponse response)
{
sillyLog.debug ("In entryOne");
if (response.getHeader ("PROFE")! = null)
{
sillyLog.debug ("Header contains PROFE:" + response.getHeader ("PROFE"));
return entryTwo (response);
}
return "returning by function entryOne \ r \ n" +
sillyLog.getMessage ();
}
@GetMapping ("two")
public String entryTwo (HttpServletResponse response)
{
sillyLog.debug ("In entryTwo");
if (response.getHeader ("PROFE")! = null)
sillyLog.debug ("Header contains PROFE:" + response.getHeader ("PROFE"));
return "returning by function entryTwo \ r \ n" +
sillyLog.getMessage ();
}
@GetMapping ("three")
public String entryThree ()
{
sillyLog.debug ("In entryThree");
return "returning by function entryThree \ n" +
sillyLog.getMessage ();
}
@GetMapping ("redirected")
public String entryRedirect (HttpServletRequest request)
{
sillyLog.debug ("In redirected");
return "returning by function entryRedirect \ n" +
sillyLog.getMessage ();
}
}
`` ''

The `entryOther` function will capture all the ** GET ** requests that go to some URI that we do not have explicitly defined. In the `entryOne` function, requests of type ** GET ** that go to the URL http: // localhost: 8080 / one or http: // localhost: 8080 / and so on will be processed.
The `sillyLog` class is a class where we will simply add log lines and then return them in the * body * of the response, so that we can see where our request has passed.
Three filters are defined in this application: `MyFilter.java`,` OtherFilter.java` and `CakesFilter.java`. The first takes precedence over the second, as it is thus established in the tag parameter ** @ Order **. Of the third I speak at the end of the article.
In the `MyFilter.java` file we define our first filter.

java
@Component
@Order (1)
public class MyFilter implements Filter {
@Autowired
SillyLog sillyLog;

@Override
public void doFilter (ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
HttpServletRequest httpRequest = (HttpServletRequest) request;
HttpServletResponse myResponse = (HttpServletResponse) response;
sillyLog.debug ("Filter: URL"
+ "called:" + httpRequest.getRequestURL (). toString ());

if (httpRequest.getRequestURL (). toString (). endsWith ("/ one")) {
myResponse.addHeader ("PROFE", "FILTERED");
chain.doFilter (httpRequest, myResponse);
return;
}
        if (httpRequest.getRequestURL (). toString (). endsWith ("/ none")) {
            myResponse.setStatus (HttpStatus.BAD_GATEWAY.value ());
myResponse.getOutputStream (). flush ();
myResponse.getOutputStream (). println ("- I don't have any to tell you -");
            return; // I'm doing nothing.
        }
if (httpRequest.getRequestURL (). toString (). endsWith ("/ redirect")) {
myResponse.addHeader ("PROFE", "REDIRECTED");
myResponse.sendRedirect ("redirected");
chain.doFilter (httpRequest, myResponse);
return;
}
if (httpRequest.getRequestURL (). toString (). endsWith ("/ cancel")) {
myResponse.addHeader ("PROFE", "CANCEL");
myResponse.setStatus (HttpStatus.BAD_REQUEST.value ());
myResponse.getOutputStream (). flush ();
myResponse.getOutputStream (). println ("- Output by filter error -");
chain.doFilter (httpRequest, myResponse);
return;
}
chain.doFilter (request, response);
}
}
`` ''
The `OtherFilter` class is simpler:

java
@Component
@Order (2)
public class OtherFilter implements Filter {
@Autowired
SillyLog sillyLog;

@Override
public void doFilter (ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
HttpServletRequest httpRequest = (HttpServletRequest) request;
HttpServletResponse myResponse = (HttpServletResponse) response;
sillyLog.debug ("OtherFilter: URL"
+ "called:" + httpRequest.getRequestURL (). toString ());
if (myResponse.getHeader ("PROFE")! = null)
{
sillyLog.debug ("OtherFilter: Header contains PROFE:" + myResponse.getHeader ("PROFE"));
}
chain.doFilter (request, response);
}

}
`` ''

The first thing we need to do to define a * general * filter is to tag the class with ** @ Component **. Then we must implement the `Filter` interface. We could also extend the `OncePerRequestFilter` class which implements the` Filter` interface and adds certain functionality so that a filter only runs once per run. In this example we are going to simplify it as much as possible and directly implement the `Filter` interface.
The `Filter` interface has three functions.
- `void init (FilterConfig filterConfig) throws ServletException`

This function will be executed by the web container. In other words: this function is only executed once, when the component is instantiated by ** Spring **.

- `void doFilter (ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException`

  This function will be executed every time an HTTP request is made. This is where we can see the content of the HTTP request, in the `ServletRequest` object and modify the response in the` ServletResponse` object. `FilterChain` is what we should execute if we want to continue the request.

- `void destroy ()`

  This function is called by the Spring web container to indicate to the filter that it will no longer be active.

As I mentioned earlier, the ** @ Order ** tag will allow us to specify the order in which the filters will be executed. In this case, this filter will have the value 1 and the next will have the value 2, so `MyFilter` will be executed before` OtherFilter`.

The `MyFilter` class performs different actions depending on the called URL. The `OtherFilter` class only adds a log when passing it.

In the example code we only use the `doFilter` function. In it, first, we convert the `ServletResponse` class to` HttpServletResponse` and the `ServletRequest` class to` HttpServletRequest`. This is necessary in order to access certain object properties that would not otherwise be available.

I am going to explain step by step the different cases contemplated in the `MyFilter` class, depending on the invoked URL.

- ** / one **: We add a ** PROFE ** header with the value ** FILTERED ** to the response. It is important to emphasize that we can only modify the response, the request is unalterable.

   Then we execute the `doFilter` function of the` chain` class, which will continue the flow of the request. In this case, the second filter would be executed and then it would be passed to the controller's `entryOne` function, where we could see that there is a * header * with the value ** PROFE **, so the function` entryTwo is called `.

  A call to this URL would return the following:

  `` ''
  > curl -s http: // localhost: 8080 / one
  returning by function entryTwo
  SillyLog: 15eb34c2-cfac-4a27-9450-b3b07f44cb50 / 1 Filter: URL called: http: // localhost: 8080 / one
  SillyLog: 15eb34c2-cfac-4a27-9450-b3b07f44cb50 / 2 OtherFilter: URL called: http: // localhost: 8080 / one
  SillyLog: 15eb34c2-cfac-4a27-9450-b3b07f44cb50 / 3 OtherFilter: Header contains PROFE: FILTERED
  SillyLog: 15eb34c2-cfac-4a27-9450-b3b07f44cb50 / 4 In entryOne
  SillyLog: 15eb34c2-cfac-4a27-9450-b3b07f44cb50 / 5 Header contains PROFE: FILTERED
  SillyLog: 15eb34c2-cfac-4a27-9450-b3b07f44cb50 / 6 In entryTwo
  SillyLog: 15eb34c2-cfac-4a27-9450-b3b07f44cb50 / 7 Header contains PROFE: FILTERED
  
  `` ''

  The first line is returned by the `entryTwo` function. The logs added are shown below.
  It is best to look at the source code if it is not clear where so many lines come from ;-)
- ** / redirect ** We add a ** PROFE ** header with the value ** REDIRECTED ** to the * response *. Then we specify that a redirect to the `redirected` URL should be included with the` myResponse.sendRedirect` statement. Finally we run the `doFilter` function so the second filter will be processed and the` entryOther` function will be called since we have no defined entry point for * / cancel *.
  This is the output that we will have if we make a request with ** curl: **

  curl
  > curl -s http: // localhost: 8080 / redirect
  
  `` ''
  
  Indeed, there is no way out. Why?. Well, because we have included a * redirected * directive and ** curl ** by default it doesn't follow those directives, which simply doesn't show anything.
  
    Let's see, what is happening adding to ** curl ** the parameter -v (verbose)
  
    curl
    curl -v -s http: // localhost: 8080 / redirect
    * Trying :: 1 ...
    * TCP_NODELAY set
    * Connected to localhost (:: 1) port 8080 (# 0)
    > GET / redirect HTTP / 1.1
    > Host: localhost: 8080
    > User-Agent: curl / 7.60.0
    > Accept: * / *
    >
    <HTTP / 1.1 302
    <PROFE: REDIRECTED
    <Location: http: // localhost: 8080 / redirected
    <Content-Length: 0
    <Date: Thu, 13 Jun 2019 13:57:44 GMT
    <
    * Connection # 0 to host localhost left intact
    `` ''
  
    This is something else, right? Now it shows in the header our value for ** PROFE ** And we see the command to redirect to http: // localhost: 8080 / redirected. Note that the HTTP code is 302, which is * redirect *.
  
    If we tell ** curl ** to follow the redirect, passing it the parameter ** - L **, we'll see what we expected.
  
    `` ''
    > curl -L -s http: // localhost: 8080 / redirect
    returning by function entryRedirect
    SillyLog: dcfc8b09-84a4-40a1-a2d6-43340abdf50c / 1 Filter: URL called: http: // localhost: 8080 / redirected
    SillyLog: dcfc8b09-84a4-40a1-a2d6-43340abdf50c / 2 OtherFilter: URL called: http: // localhost: 8080 / redirected
    SillyLog: dcfc8b09-84a4-40a1-a2d6-43340abdf50c / 3 In redirected
    
    `` ''
  
    Well, almost what we expected. Note that there have been two HTTP requests to our service and only the data of the second is shown.
  
  - ** / none **. I set the HTTP code to return to BAD_GATEWAY and in the body I put the text * "I don't have any to tell you" *. I don't run the `doFilter` function so neither the second filter will be called, nor would it be passed to the controller.
  
    curl
    > curl -s http: // localhost: 8080 / none
    - I don't have any to tell you -
    `` ''
  
  - ** / cancel **. I set the HTTP code to return to BAD_REQUEST and in the body I put the text * "Output by filter error" *. I run the `doFilter` function so the` OtherFilter` filter will be run and pass through the controller's `entryOther` function, since we have no defined entry point for * / cancel *
  
    curl
    > curl -s http: // localhost: 8080 / cancel
    - Output by filter error -
    returning by function entryOther
    SillyLog: 1cf7f7f9-1a9b-46a0-9b97-b8d5caf734bd / 1 Filter: URL called: http: // localhost: 8080 / cancel
    SillyLog: 1cf7f7f9-1a9b-46a0-9b97-b8d5caf734bd / 2 OtherFilter: URL called: http: // localhost: 8080 / cancel
    SillyLog: 1cf7f7f9-1a9b-46a0-9b97-b8d5caf734bd / 3 OtherFilter: Header contains PROFE: CANCEL
    SillyLog: 1cf7f7f9-1a9b-46a0-9b97-b8d5caf734bd / 4 In entryOther
    SillyLog: 1cf7f7f9-1a9b-46a0-9b97-b8d5caf734bd / 5 Header contains PROFE: CANCEL
    
    `` ''
  
    Note that the body added in the filter is older than what was returned by the controller.
  
  - ** Others ** In any other call, the `doFilter` function of the` chain` class will be invoked, so it will go to the next filter and then to the appropriate controller function.
  
    curl
    > curl -L -s http: // localhost: 8080 / three
    returning by function entryThree
    SillyLog: a2dd979f-4779-4e34-b8f6-cae814370426 / 1 Filter: URL called: http: // localhost: 8080 / three
    SillyLog: a2dd979f-4779-4e34-b8f6-cae814370426 / 2 OtherFilter: URL called: http: // localhost: 8080 / three
    SillyLog: a2dd979f-4779-4e34-b8f6-cae814370426 / 3 In entryThree
    `` ''
  
  To specify that a filter is only active for certain URLs, you must register it explicitly and not mark the class with the tag ** @ Component **. In the example project in the `FiltersApplication` class we see the function where a filter is added:
  
  `` ''
  @Bean
  public FilterRegistrationBean <CakesFilter> cakesFilter ()
  {
  FilterRegistrationBean <CakesFilter> registrationBean = new FilterRegistrationBean <> ();
  registrationBean.setFilter (new CakesFilter ());
  registrationBean.addUrlPatterns ("/ cakes / *");
  return registrationBean;
  }
  `` ''
  
  The `CakesFilter` class is as follows:
  
  `` ''
  @Order (3)
  public class CakesFilter implements Filter {
  @Override
  public void doFilter (ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
  HttpServletResponse myResponse = (HttpServletResponse) response;
  myResponse.addHeader ("CAKE", "EATEN");
  chain.doFilter (request, response);
  }
  }
  `` ''
  
  When making a call to a url that begins with * / cakes / \ ** we will see how the last filter is executed.
  
  `` ''
   curl -s http: // localhost: 8080 / cakes
  returning by function entryOther
  SillyLog: 41e2c9b9-f8d2-42cc-a017-08ea6089e646 / 1 Filter: URL called: http: // localhost: 8080 / cakes
  SillyLog: 41e2c9b9-f8d2-42cc-a017-08ea6089e646 / 2 OtherFilter: URL called: http: // localhost: 8080 / cakes
  SillyLog: 41e2c9b9-f8d2-42cc-a017-08ea6089e646 / 3 In entryOther
  SillyLog: 41e2c9b9-f8d2-42cc-a017-08ea6089e646 / 4 Header contains CAKE: EATEN
  
  `` ''
  
  Because of the way ** Spring ** handles its context variables, it is not possible to inject the `SillyLog` object with a ** @ Autowired **. If we inject it we will see how the variable has the value ** null **
  And with this I end this post. Until next time !!
