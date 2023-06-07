---
layout: post
title:  "Enforcing single session"
date:   2023-05-11 00:00:00 +0800
categories: spring security
---

Spring security can be configured to timeout when the user leaves the the session idle for too long.
However, if the page has ajax polling, the calls to poll update the last request time,
so the session is never timed out.

One approach to fix this is to identify the ajax poll calls, and don't update the last request time.

In the application.yaml, configure the session timeout
{% highlight %}
server:
  servlet:
    session:
      timeout: 3m
{% endhighlight %}

In the security configuration, register the custom session registry
{% highlight java %}
@Configuration
@EnableWebSecurity
public class SecurityConfig {
...

  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    ...
    http.sessionManagement(
        sm -> sm
        .maximumSessions(1)
        .sessionRegistry(getSessionRegistry())
        .expiredUrl("/public/sessionTimeout.xhtml"));
    ...
  }

  @Bean
  public SessionRegistry getSessionRegistry() {
    return new CustomSessionRegistryImpl();
  }
  
}
{% endhighlight %}

In the session registry, identify ajax calls, then return without updating the last request time.

{% highlight %}
<p:poll interval="5" ... update="poll_ui_element"/>
{% endhighlight %}

{% highlight java %}
@Component
public class CustomSessionRegistryImpl extends SessionRegistryImpl {
...
 @Override
  public void refreshLastRequest(String sessionId) {
    // Check if it's an ajax call
    HttpServletRequest request =
        ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();

    String ajaxHeader = ((HttpServletRequest) request).getHeader("X-Requested-With");
    final HttpServletRequest req = (HttpServletRequest) request;
    if ("XMLHttpRequest".equals(ajaxHeader)) {
      // Now see if it's the polling
      try {
        ContentCachingRequestWrapper servletRequest = new ContentCachingRequestWrapper(req);
        servletRequest.getParameterMap(); // needed for caching!!

        String read = ByteSource.wrap(servletRequest.getContentAsByteArray())
            .asCharSource(StandardCharsets.UTF_8).read();
        if ((read != null) && read.contains("poll_ui_element")) {
          return;
        }
      } 
      ...
    }

    // SessionId not changed, refresh the last request time as usual
    super.refreshLastRequest(sessionId);
  }
}
{% endhighlight %}
