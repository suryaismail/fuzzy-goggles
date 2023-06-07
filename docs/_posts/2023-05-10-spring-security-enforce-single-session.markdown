---
layout: post
title:  "Enforcing single session"
date:   2023-05-10 00:00:00 +0800
categories: spring security
---

Spring security tracks sessions per browser.  So if a user logs in from 2 separate browsers,
a new session is created for each one. Spring security does not prevent multiple sessions.

To get this behaviour, we overwrite the SessionRegistryImpl. First register the 
custom session registry in the security configuration.

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
        .sessionRegistry(getSessionRegistry())
    ...
  }

  @Bean
  public SessionRegistry getSessionRegistry() {
    return new CustomSessionRegistryImpl();
  }
  
}
{% endhighlight %}

Session information needs to be stored in a database (or a cache). Every time a user accesses a page that requires authentication, the session registry is called to update the last request time.
We take advantage of that behaviour to check if the sessionId for a user is changed between request times.

If the sessionId changes, either the new session or old session can be invalidated. This enforces a single session per user.

{% highlight java %}
@Component
public class CustomSessionRegistryImpl extends SessionRegistryImpl {
...
 @Override
  public void refreshLastRequest(String sessionId) {
    // Retrieve session information from DB
    SessionInformation sessionInformation = getSessionInformation(sessionId);
    String username = sessionInformation.getPrincipal();
    String dbSessionId = sessionDB.getSession(userName);

    // Check if this is still the current session
    if (!sessionId.equals(dbSessionId)) { // session Id changed
      // Expire this session
      getSessionInformation(sessionId).expireNow();
      return;
    }

    // SessionId not changed, refresh the last request time as usual
    super.refreshLastRequest(sessionId);
  }
}
{% endhighlight %}
