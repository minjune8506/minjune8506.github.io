---
layout: post
title: hibernate-validation
date: 2025-12-03 20:01 +0900
categories: [backend]
tags: [validation]
---

# default message interpolation

- Constraint Violation messages are retrieved from so called **message descriptors**.
- each constraint defines its default message descriptor using the **message attribute**
- At declaration time, *the default descriptor can be overriden with a specific value*
- If a constraint is violated, its descriptor will be **interpolated** by the validation engine using the currently configured `MessageInterpolator`
- interpolated error message can then be retirieved by calling `ConstraintViolation#getMessage()`
- Message descriptors can contain **message parameters** as well as **message expressions** which will be resolved during interpolation.
- Message paramters are string literals enclosed in `{}`
- Message expressions are string literals enclosed in `${}`

## interpolation algorithm

- resolve any message parameters by using them as key for the resource bundle *ValidationMessages*
  - if this bundle contains an entry for a given message parameter, that parameter will be replaced in the message with the corresponding value from the bundle.
  - this step will be executed recursively in case the replaced value again contains message parameters
  - the resource bundle is exepected to be provided by the application developer
    - by adding a file named *ValidationMessages.properties* to the classpath
  - can also create localized error messages by providing locale specific variations of this bundle
    - by adding a file named *ValidationMessages_en_US.properties* to the classpath
    - by default, the JVM's default locale(`Locale#getDefault()`) will be used when looking up messages in the bundle.
- resolve any message parameters by using them as key for a resource bundle containing the standard error messages for the built-in constraints as defined in Appendix B of the Jakarta Validation specification.
  - in the case of Hibernate Validator, the bundle is named `org.hibernate.validator.ValidationMessages`
  - if this step triggers a replacement, step 1 executed again
- resolve any message parameters by replacing them with the value of the *constraint annotation memeber of the same name*
  - this allows to refer to attribute values of the constraint (e.g. `Size#min()`) in the error message (e.g. `must be at lest ${min}`)
- resolve any message expressions by evaluating them as expressions of the **Unified Expression Language**.

## special characters

`{`, `}`, `$` have a special meaning in message descriptors

if you want to use them literally, they need to be escaped to `\{`, `\}`, `\$`, `\\`

## interpolation with message expressions

- it is possible to use the **Jakarta Expression Language** in constraint violation messages
- this allows to define error messages based on *conditional logic* and also enables *advanced formatting* options

the validation engine makes the following objects available in the Expression Language context

- the attribute values of the constraint mapped to the attribute names
- the currently validated value under the name **validatedValue**
- a bean mapped to the name formatter exposing the var-arg method `format(String format, Object... args)` which behaves like `java.util.Formatter.format(String format, Object ...args)`

### ExpressionLanguageFeatureLevel

you can define the Expression Language feature level when bootstrapping the ValidatorFactory.

- `NONE`: Expression Language interpolation is fully disabled
- `VARIABLES`: allow interpolation of the variables injected via `addExpressionVariable()`, resources bundle and usage of the `formatter` object
- `BEAN_PROPERTIES` (default): allow everything `VARIABLES` allows plus the interpolation of bean properties
- `BEAN_METHODS`: Also allow execution of bean methods. Can be considered safe for hardcoded constraint messages but not for custom violations where extra care is required.
