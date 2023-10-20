---
layout: post
title: MySQL Time Data Type
date: YYYY-MM-DD HH:MM:SS +09:00
categories:
  - Database
  - MYSQL
tags:
  - Time Data Type
---

# MySQL Time Data Type

MySQL의 날짜 / 시간 타입은 DATE, DATETIME, TIME, TIMESTAMP가 있다.

## DATE

날짜만 포함하는 타입

YYYY-mm-dd 형식으로 입력 가능

1000-01-01 ~ 9999-12-31

## DATETIME

날짜와 시간을 모두 포함하는 타입

YYYY-mm-dd HH:MM:SS 형식으로 입력 가능

1000-01-01 00:00:00 ~ 9999-12-31 23:59:59

## TIME

시간만 포함하는 타입

HH:MM:SS 형식으로 입력 가능

-838:59:59 ~ 838:59:59

하루는 24시간인데 왜 838까지 표현가능한지 이유는 24시간이 초과하는 시간을 나타내고 싶을때도 있기 때문이다.

예를 들어 26시간 뒤에 실행시켜야 하는 기능을 표현할 수 있다.

## TIMESTAMP

날짜와 시간을 모두 포함하는 타입

YYYY-mm-dd HH:MM:SS 형식으로 입력 가능

1970-01-01 00:00:01 ~ 2038-01-19 03:14:07 UTC 까지 표현 가능

### TIMESTAMP vs DATETIME

TIMESTAMP는 DATETIME과 달리 TIMEZONE에 영향을 받는다.

데이터베이스의 TIME_ZONE 시스템 변수에 입력된 시간대 정보를 기반으로 데이터를 입력받아 UTC로 변환하여 저장한다.

TIME_ZONE이 변경되면 변경된 시간을 보여준다는 특징이 있다.

TIMESTAMP(6)의 경우 밀리초까지 표현 가능하다.

6은 밀리초를 몇자리까지 표현할 것인지 결정한다.

년도가 2038까지인 이유는 유닉스 시간에 32비트 정수형을 쓰는데, 2038년이 지나면 부호비트가 1로 바뀌면서 오버플로우가 일어나기 떄문이다.

MYSQL은 새로운 버전으로 이 문제를 해결했고, MariaDB는 아직 이 문제를 해결한 버전을 내놓지 않은 것으로 보인다.

[Y2K38 문제](https://medium.com/finda-tech/mysql-timestamp-%EC%99%80-y2k38-problem-d43b8f119ce5)