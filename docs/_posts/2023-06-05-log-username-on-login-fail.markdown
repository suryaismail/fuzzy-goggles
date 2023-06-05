---
layout: post
title:  "Logging username when login fails"
date:   2023-06-05 23:33:54 +0800
categories: spring security
---

Spring security provides the ability to create a custom AuthenticationFailureHandler.
However, by the time the we get to this handler, the principle is set to null.
The username is not available.

It's sometimes useful to log the username used when authentication fails.

public class CustomApplicationListener implements ApplicationListener<AuthenticationFailureBadCredentialsEvent> {
