---
layout: post
title: Spring, Controller, Service, Repository
date: 2022-01-24 19:00
categories: Java
tags: Java
toc: true
toc_sticky: true
---

![image](https://user-images.githubusercontent.com/96677719/150688504-7f6d3050-6d0f-40aa-9e91-88a4485f044a.png)


![image](https://user-images.githubusercontent.com/96677719/150688464-f84d46d3-7443-4c0e-bc75-d1aa02943631.png)

![image](https://user-images.githubusercontent.com/96677719/150688446-3e2afb0e-2e49-41cc-a2d0-994d5cc3ea9d.png)


###### ๐Controller
์ฃผ์๋ฅผ ์ฐพ์์ ์ฌ์ฉ์์ ์์ฒญ์ ๋ง๊ฒ ๋ฐ์ดํฐ๋ฅผ ์ ๋ฌํ๋ค. ๋ ์์ธํ ๋งํ์๋ฉด, ์ฌ์ฉ์์ request๋ฅผ ์ฒ๋ฆฌํ ํ ์ง์ ๋ ๋ทฐ(html, jsp ๋ฑ ํ๋ฉด์ ๋์ฐ๋ ํ์ผ)์ ๋ชจ๋ธ ๊ฐ์ฒด(Object, Dto ์ฌ์ฉ์๋ก๋ถํฐ ๋ฐ์ ๋ฐ์ดํฐ)๋ฅผ ๋๊ฒจ์ฃผ๋ ์ญํ ์ ํ๋ค. 

###### ๐Service
ํ์ ๊ฐ์, ์กฐํ ๋ฑ ๋น์ฆ๋์ค๋จ์ ๊ธฐ๋ฅ๋ค ํน์ ์ค๋ณต ๊ฒ์ฌ ๋ฑ์ด ์ํ๋๋ค. DB์์ ์ ์ ์๊ฒ, ํน์ ๊ทธ ๋ฐ๋๋ก ๋ฐ์ดํฐ๋ฅผ ๊ฐ๊ณตํ์ฌ ์ ๋ฌํ๋ ์ญํ ์ ํ๋ค. repository๋ฅผ ๊ฑฐ์น๊ธฐ ๋๋ฌธ์ ์ง์  DB์ ์ ๊ทผํ์ง๋ ์๋๋ค.

###### ๐Repository
์ ์ฅ์๋ผ๋ ์๋ฏธ๋ก DB์ ์ ๊ทผํ๋ ์ญํ ์ ๋งก๋๋ค. ์ผ์ข์ ์ธํฐํ์ด์ค๋ก, JpaRepository๋ฅผ ์์ ๋ฐ์ผ๋ฉด ๊ธฐ๋ณธ์ ์ธ CRUD๊ฐ ์๋์ผ๋ก ์์ฑ๋์ด ์ด๋ฅผ ์ฝ๊ฒ ์ฒ๋ฆฌํ  ์ ์๋ค. 

###### ๐์ถ์ฒ

https://snepbnt.tistory.com/312

https://it-recording.tistory.com/31?category=971893
