---
title: Spring, RestController와 ModelAndView
categories: Spring
tags: Spring
toc: true
toc_sticky: true
---

###### 📌Rest

Rest = Representational State Transfer 

하나의 URI가 하나의 리소스를 대표하게 하는 것. /sample/detailPage는 어떠한 페이지가 아닌, detailPage라는 변수가 담고 있는 데이터인 셈이다. 즉 해당 주소의 detailPage은 detailPage.html을 의미하는 것이 아니라 html이 담고 있는 데이터를 의미한다. 

###### 📌Controller vs RestController

![image](https://user-images.githubusercontent.com/96677719/150690194-71fe9dda-1e2a-426d-aa7f-dffb3dbf4eef.png)

![image](https://user-images.githubusercontent.com/96677719/150690199-c49b029e-6d63-4ce4-a34c-c1681980fbef.png)

컨트롤러를 지정하는 어노테이션은 @Controller와 @RestController가 있다. @Controller는 전통적인 MVC의 컨트롤러로 View를 반환하는데에 사용된다. Data를 반환하고자 할 때는 @ResponseBody를 사용한다. @RestController란 이러한 @Controller에 @ResponseBody를 추가한 것이다. 기능상으로는 같지만 컨트롤러 자체의 용도를 지정해준다는 점에서 변화가 있다. 주 용도는 Json 형태로 객체를 반환하는 것에 있다. 

예를 들어서

```java
@Controller
public class Example {
    @GetMapping("/detailPage")
    public String detail(){
        return "details";
    }
}
```
위와 같은 경우 화면에 details.html 뷰를 보여주고
```java
@RestController
public class Example {
    @GetMapping("/detailPage")
    public String detail(){
        return "details";
    }
}
```
위와 같은 경우에는 details 라는 String을 그대로 보여준다.

이때 return 방식은 int String json list<> dto 커스텀 클래스까지 모두 가능하다

###### 📌ModelAndView

@RestController에서 View 값을 반환하는 방법에는 ModelAndView를 사용하여 model과 view를 모두 반환하는 방법이 있다. view 중에서 jsp에서는 ModelAndView로 보낸 값 중 Model에 해당하는 부분을 바로 받아서 사용할 수 있으나 html 파일에서 받으려면 Thymleaf 등을 통한 서버 사이드 렌더링이 따로 필요하다. (Thymleaf 링크) 스프링에서는 jsp를 사용하는 대신 서버 사이드 렌더링을 사용할 것을 권장한다. 

###### 📌 출처

https://mangkyu.tistory.com/49

https://hongku.tistory.com/116

https://milkye.tistory.com/283
