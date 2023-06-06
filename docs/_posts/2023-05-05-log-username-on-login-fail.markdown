---
layout: post
title:  "Logging username when login fails"
date:   2023-05-05 00:00:00 +0800
categories: spring security
---

Spring security provides the ability to create a custom AuthenticationFailureHandler.
However, by the time the we get to this handler, the principal is set to null.
The username is not available.

It's sometimes useful to log the username used when the password is wrong.
To do this, we need to create a custom ApplicationLister for the AuthenticationFailureBadCredentialsEvent event. 

{% highlight java %}
public class CustomApplicationListener implements ApplicationListener<AuthenticationFailureBadCredentialsEvent> {
...
  @Override
  public void onApplicationEvent(AuthenticationFailureBadCredentialsEvent event) {
    Object userName = event.getAuthentication().getPrincipal();
    LOGGER.info("Failed password for user {}", userName);
      ...
    }
  }
{% endhighlight %}

