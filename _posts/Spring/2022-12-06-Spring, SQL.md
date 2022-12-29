---
title: Spring, SQL
categories: Spring
tags: Spring
toc: true
toc_sticky: true
---

## 📌 SQL이란?

Structed Query Language, 구조적 쿼리 언어. 관계형 데이터베이스 시스템 (RDBMS)에서 데이터를 정의, 조작, 제어하기 위해 사용하는 언어. 현재 SQL의 표준은 ANSI SQL로 각 DBMS 프로그램에서 이를 기반으로 개발된 개별 SQL을 사용한다. 예시로는 MySQL, SQLite, PostgroSQL 등이 있다. 

## 📌 SQL 구문의 종류

DDL: Data Definition Language, 데이터 정의 언어. **DB나 테이블** 등을 생성, 삭제하거나 구조를 변경하기 위한 명령어

- CREATE: 테이블 생성
- ALTER: 테이블 구조 변경
- DROP: 테이블 삭제

DML: Data Manipulation Langeage, 데이터 조작 언어. DB에 저장된 **데이터**를 처리, 조회, 검색하기 위한 명령어

- SELECT: 데이터 조회
- INSERT: 데이터 삽입
- UPDATE: 데이터 갱신

DCL: Data Controll Language, 데이터 제어 언어. DB에 저장된 데이터를 관리하기 위해 **데이터의 보안성, 무결성**을 제어, 관리하고 접근하는 권한을 다루기 위한 명령어

- GRANT: 권한 부여
- REVOKE: 권한 취소

## 📌 SQL의 언어적 특성

1. 대소문자를 구분하지 않는다. 단, 서버 환경이나 DBMS 종류에 따라 DB 또는 필드명에 대해 대소문자를 구분하기도 한다.
2. 반드시 ;로 끝난다.
3. 고유값은 ''로 감싸준다.
```
SELECT * FROM EMP WHERE NAME = 'James';
```
4. 객체를 나타낼 때는 ``로 감싸준다.
```
SELECT `COST`, `TYPE` FROM `INVOICE`;
```
5. --을 통해 한 줄 주석을, /* */을 통해 여러줄 주석을 나타낸다.
```
-- SELECT * FROM EMP;
```
### 출처

http://www.tcpschool.com/mysql/DB

https://velog.io/@gparkkii/DatabaseSQLNoSQL

https://edu.goorm.io/learn/lecture/15413/%ED%95%9C-%EB%88%88%EC%97%90-%EB%81%9D%EB%82%B4%EB%8A%94-sql