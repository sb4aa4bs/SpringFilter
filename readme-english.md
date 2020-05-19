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
Send feedback
History
Saved
Community
