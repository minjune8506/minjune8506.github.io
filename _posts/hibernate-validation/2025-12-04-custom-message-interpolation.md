---
layout: post
title: custom-message-interpolation
date: 2025-12-04 21:38 +0900
categories: [backend]
tags: [validation]
---

# Custom message interpolation

it is possible to plug in a custom `MessageInterpolator` implementation.

- Custom interpolators must implement the interface `jakarta.validation.MessageInterpolator`
- implementations must be thread-safe
- it is recommended that custom message interpolators delegate final implementation to the default interpolator, which can be obtained via `Configuration#getDefaultMessageInterpolator()`
- to use a custom message interpolator it must be registered either by configuring it in the Jakarta Validation XML descriptor `META-INF/validation.xml` or by passing it when bootstrapping a `ValidatorFactory` or `Validator`

## `ResourceBundleLocator`

want to retrieve error messages from other resource bundles than `ValidationMessages`

in this situation Hibernate Validator's `ResourceBundleLocator` SPI can help.

SPI: Service Provider Interface

- the default message interpolator in Hibernate Validator is `ResourceBundleMessageInterpolator`
- using an alternative bundle only requires passing an instance of `PlatformResourceBundleLocator` with the bundle name when bootstrapping the `ValidatorFactory`
-  you can implement a completely different `ResourceBundleLocator`
