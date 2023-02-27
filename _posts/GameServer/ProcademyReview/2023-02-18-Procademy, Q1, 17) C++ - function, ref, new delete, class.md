---
title: Procademy, Q1, 17) C++ - function, ref, new delete, class
categories: ProcademyReview
tags: 
toc: true
toc_sticky: true
---

이 포스트는 프로카데미 (게임 서버 아카데미) 수업을 바탕으로 공부한 내용을 정리한 것입니다. 

# **1. 함수**

## **01) 디폴트 인자**

디폴트 인자는 함수 밖에서 해당 함수를 호출할 때 값이 들어간다. 인자에 값이 들어갔는지 아닌지 함수 내부에서 알 방법이 없기 때문에 호출 시점에 결정되는 것이다. 따라서 디폴트 선언을 해도 함수 내부에서는 아무런 변화가 없다. 헤더의 선언부에서만 디폴트 선언을 하는 것도 이 때문이다. 

```c++
#include <iostream>

void Attack(int Type)
{
	switch(Type)
	{
		case 1:
		case 2:
		…
		default:
	}
}
```

이렇게 만든 상황에서 기존 함수 호출 부분은 그대로 가되, 새로 추가되는 case에만 필요한 파라미터가 있다면 아래처럼 사용할 수 있다. 하위호환성을 유지하면서 추가 변수를 사용할 수 있는 것이다. 

```c++
#include <iostream>

void Attack(int Type, int Attack = 0)
{
	switch(Type)
	{
		case 1:
		case 2:
		…
        case 10:
            if(Attack != 0)
            {
                //TO-DO
            }
		default:
	}
}
```

## **02) 인라인 함수**

```c++
#include <iostream>

inline void Attack(int Type)
{
	int a = 0;
	a += Type;
	printf(“%d”, a);
}
```

이 문법을 사용하기 위해서는 속성을 수정해주어야 한다. 최적화는 사용 안함으로 하고 최적화 아래에 인라인 함수 확장만 inline 확장으로 하면, inline 표시한 것만 inline 처리할 수 있게 된다. 확장 확대를 할 경우 컴파일러 최적화가 켜졌을 때와 유사하게 inline 처리된다. stl의 경우 처음부터 최적화 컴파일러를 가정하고 만든 것이라, 간단한 것도 함수로 여러번 감싸고 있기 때문에 inline을 꺼두고 할 경우 stl에서 속도 저하가 많이 일어날 수 있다.  따라서 컴파일러 최적화를 끄고 오류 날 일이 없는 간단한 함수의 경우에만 inline 확장 처리 하는 것을 권장한다. 


# **2. 동적 할당**

## **01) new - delete와 객체**

new는 malloc을, delete는 free를 랩핑한 함수로, 연산자지만 함수 호출과 연산자의 중간쯤 되는 성질을 가진다. 원시형 타입을 new-delete 할 때는 malloc을 호출하는 함수처럼 동작한다. 반면 클래스 타입을 new-delete 하면 생성자, 소멸자가 호출되며 함수이자 연산자처럼 동작한다. new-delete를 연산자 오버로딩 하게 될 경우, 기존 new-delete에서 생성자, 소멸자 영역을 제외하고 함수 역할, 즉 malloc 역할만 가져올 수 있게 된다. 

```c++
int* p = new int[10];
p[0] = 0;
p[1] = 1;
delete p; 
```

이 경우 p 배열 전체가 delete 되지 않고 맨 앞 p[0]만 delete 하게 될 거라고 추정된다. 

```c++
int* p = new int;
p[0] = 0;
delete [] p;
```

이 경우에는 반대로 p가 배열이 아님에도 배열을 delete 하고 있으므로 문제가 날 것으로 추정된다. 그러나 실제로는 문제 없이 잘 실행이 된다. new - delete가 malloc-free를 랩핑하고 있기 때문에 알아서 p 전체를 free 해주게 된다. 

```c++
class CTest
{
public:
	int x;
	int y;
};

int main()
{
	CTest* p = new CTest[10];
    p[0].x = 0;
    p[1].y = 1;
    delete p; 

	CTest* p = new CTest;
    p[0].x = 0;
    delete[] p; 
}
```

위 예시처럼 new - delete의 대상이 객체가 되어도 동일하게 잘 동작한다. 그러나 해당 객체가 생성자, 소멸자를 갖고 있는 경우 문제가 생긴다. 보통 쉬운 이해를 위해 컴파일 과정에서 컴파일러가 자동으로 생성자, 소멸자를 만들어준다고 설명하나, 실제로는 명시적으로 만들지 않은 생성자, 소멸자를 자동으로 만들어주지는 않는다. 따라서 위 예시도 문제 없이 동작할 수 있는 것이다.

반면 생성자 있을 경우, new 하는 지점에서 어셈블리를 까보면 vector constructor iterator이라는 함수를 호출하는 것을 볼 수 있다. 기존에는 40 크기의 힙을 한번에 malloc 했다면, 이 경우에는 배열 개수만큼 4byte씩 반복문을 돌면서, 매 배열인자마다 생성자를 호출하는 것이다. 만약 생성자만 만들고 소멸자를 만들지 않았다면 이 경우에도 크게 문제 없이 동작한다. 

그러나 소멸자를 적은 경우 어셈블리가 조금 다르게 나온다. 확인해보면 소멸자가 있는 new 배열이 호출될 때는 내부에서 더 많은 스택을 잡는 것을 볼 수 있는데, 이는 소멸자가 호출되어야 하는 횟수, 배열의 크기를 기록해주기 위한 공간이다. 확인해보면 우리에게 넘겨주는 주소 바로 앞에 해당 횟수를 적어둔 것을 볼 수 있다. 이러한 이유로 delete[]를 호출할 경우 횟수를 적어둔 부분도 함께 해제하기 위해서 우리에게 던져진 주소보다 한칸 앞 주소부터 해제하는 반면, delete를 호출할 경우 우리에게 던져진 주소부터 해제한다. 

따라서 이처럼 호출해야 할 소멸자가 있는 경우에는 delete[]와 delete를 잘 구분해주어야 한다. 만약 delete[]를 써야하는데 delete를 쓴다면 heap이 깨져서 크래시가 난다. heap은 동적 할당을 할 때 할당한 지점을 기억하고 있다가, 해제해야 하는 지점과 할당한 지점이 다를 경우 크래쉬를 터뜨리기 때문이다. delete[]는 delete보다 한주소 앞을 해제하기 때문에 문제가 생긴다. 소멸자 역시 당연히 배열 형태가 아니기 때문에 하나만 실행되고 이후 배열에 들어있는 객체에서는 실행되지 않을 것이다. 

반대로 delete를 해야하는데 delete[]를 한다면 개수를 넣는 포인터 바로 앞 주소를 잘못 참조하게 된다. 이 경우 앞에 그 위치에 무슨 값이 들어있는 지에 따라 다른 오류가 난다. fdfdfdfd… 와 같이 어마어마하게 큰 값이 들어있었을 경우 소멸자가 계속 호출되게 된다. 만약 소멸자에서 멤버를 건드리게 된다면 잘못된 메모리 참조가 일어날 것이고, printf만 한다면 그냥 printf만 계속 일어난다. 만약 둘 다 사용할 경우, 소멸자가 뒤에서부터 앞으로 진행되기 때문에 printf 없이 잘못된 메모리 참조가 바로 뜨게 된다. 과거에는 앞에서부터 뒤로 소멸자가 호출됐지만 현재에는 안전 문제로 뒤에서부터 앞으로 소멸자가 호출되는 것으로 수정되었다. 

delete에서 에러가 났다면 이를 참고하여 살펴보는 것이 좋다. delete[]를 해야되는데 delete를 하는 경우 heap에서 크래시를 내고, delete를 해야되는데 delete[]를 하는 경우에는 보통 소멸자에서 크래시를 낸다. 굳이 이 경우가 아니더라도, 객체를 쓰는데 크래시가 난다면 this가 어디를 가리키고 있는지, 그 곳의 값이 존재할 수 있는 값인지 먼저 찍어보는 것이 좋다. 

## **02) delete와 댕글링 포인터**

```c++
int* p = new int;
delete p;
*p = 0;
```

이러한 실험을 malloc 배울 때도 했었다. 해제 뒤에 저렇게 사용하는 포인터를 두고 댕글링 포인터라고 부른다. malloc의 경우 위 상황에서 예외를 내지 않고 그냥 넘어가지만, new - delete의 경우 예외를 터뜨린다. delete 되는 변수에 0x8123을 넣고, 사용한 포인터가 0x8123을 가리킬 경우에 예외를 내는 것이다. 예전에 배웠다시피 0x0000 - 0xffff는 사용자가 접근할 수 없는 주소로 지정되어 있는데, 그 주소값을 포인터에 넣어줌으로써 댕글링 포인터를 사용하는 상황을 막아주는 것이다. 이는 Visual Studio 2023부터 추가된 기능으로 추정되며, Visual C 차원에서 해주는 것이라 다른 컴파일러를 사용할 때는 해당 예외가 나지 않을 수 있다.  

## **03) placement new**

```c++
char* p = new char[4];
int* p2 = new (p) int;
```

이처럼 포인터에 new ( 포인터 ) 를 할 경우, 새로운 메모리를 할당하는 대신 기존에 할당 받았던 포인터를 반환해준다. 이 코드는 p에서 할당받았던 영역을 int처럼 사용하겠다는 의미가 되므로, char[4]를 int로 캐스팅 하는 코드가 될 것이다. 

```C++
char* p = (char*) malloc(sizeof(CTest));
CTest* p2 = new (p) CTest;
```

반면 위 경우처럼 만약 class 포인터를 랩핑한다면 추가적인 메모리 할당을 하지 않고 괄호 안에 있는 클래스의 생성자만을 호출하는 코드가 된다. 소멸자는 사용자가 직접 호출할 수 있는 반면 생성자는 그게 불가능하기 때문에, 할당 받는 시점과 생성자 호출 시점을 분리하고 싶다면 위와 같이 사용할 수 있다. 메모리 풀과 같이 이미 할당 받은 메모리를 다르게 재사용하는 상황 등에서 유용하게 사용할 수 있다. 


# **3. 객체지향**

## **01) this**

우리가 객체지향적이지 않은 방식으로 구조체 변수를 건들여야 한다고 하면 아래와 같은 방식을 사용할 것이다. 

```c++
struct Test
{
	int _a = 0;
}
void AAA(Test* p)
{
	p->_a = 0;
}
```
```c++
class Test
{
    void AAA()
    {
        _a = 0; // this -> _a = 0;
    }

	int _a = 0;
}
```
이 둘은 사실상 동일하게 동작한다. (this->)는 생략해서 적어도 되지만, 어쨌든 내부적으로는 해당 변수 this의 주소를 참조하여 값을 바꾸는 것이다. 

아래 코드의 디스어셈블리를 까보면 AAA라는 멤버 함수를 호출하기 전에 mov ecx [p]를 통해 자기 자신의 주소를 저장해두고, AAA 멤버 함수 내에서 mov this ecx를 통해 이 p 값을 전달한다.  

```c++
class Test
{
    void TT(int a)
    {
        printf(“%d”, a);
    }

	int _a = 0;
}

int main()
{
    CTest* p = nullptr;
    p->TT(10);
}
```
내부적으로 보면 ecx에 nullptr을 넣고 TT라는 전역 함수를 호출하는 것과 동일하게 동작할 것이다. TT 내부에서 this의 멤버 변수를 건드린다면 예외가 나겠지만, 위 코드의 경우 멤버 변수를 건드리지 않기 때문에 예외가 나지 않을 것이다. 조사식에서 this를 치면 함수 내부에서 this가 어떤 주소값을 갖고 있는지 확인할 수 있다. 멤버 함수에서 예외가 난다면 이를 먼저 확인해보는 것이 좋다. 이때 this 포인터 자체는 멤버 함수의 호출 함수가 아니라 멤버 함수 내 지역 변수로서 존재한다. 위에서 디스어셈블리로 확인했듯이 매개변수로 p 값을 받아서 멤버 함수의 스택 내 this에 전달하는 것이다. 이 말은 즉슨, stack 내에 저장된 값으로 멤버를 건드리고 있기 때문에 내 실수로 인해 stack 내의 this가 오염될 수 있다는 것을 의미한다. 

## **02) 멤버 변수의 위치**

일부 책에서는 public이 외부에서도 쓸 수 있는 것이니 위에 올리기를 권장한다. 클래스에서 멤버 함수와 변수를 어떻게 배치하는지는 Cache hit, Cache miss와도 연관되므로 혼자 코드를 만들 때는 참고하는 것이 좋다. 자주 사용하는 것들은 가까이 붙여놓는 게 hit 비율이 높아지기 때문이다. 하지만 유지보수를 들어가면 실제로는 지켜지지 않는다. 가독성 때문에 일반적으로는 업데이트 시에 주석 치고 컨텐츠, 주석 단위로 달게 된다.

## **03) Getter - Setter**

```c++
class Player
{
pubilc: 
	int GetX() { return _x; }
    void SetX(int x) { _x = x; }

private:
	int _x;
	int _y;
	int _id;
	int _hp;
}
```
위와 같은 사용은 객체 지향의 원리에 어긋난다. 객체 지향은 자신의 멤버를 객체 내부에서 통제하도록 한다는 것인데, Getter-Setter가 있다는 것 자체가 외부에서 멤버를 건드리겠다는 의미가 된다. Position, Point와 같이 어쩔 수 없이 사용해야 하는 객체도 있겠지만 되도록 지양하는 것이 좋다. 위치를 바꾸고 싶다면 Player.Move()를 해서 x를 내부에서 건드리도록 하고, hp를 줄이고 싶다면 Player.Damaged 를 해서 hp를 내부에서 건드리는 것이 좋다. 

**+)** 과거에는 멤버변수라는 뜻으로 m_ 를 많이 붙였는데 최근에는 _ 만 붙이는 추세라고 한다. 

## **04) class 내의 const 함수**

const 함수는 멤버 변수의 값을 변경하지 못하며 const 함수는 const 가 아닌 함수를 호출할 수 없다. 간접적인 멤버의 변경 가능성까지 막는 것이다. 쓰기가 불가능한 참조를 줄 경우 유용하게 사용할 수 있다. 

## **05) Initialize** 

멤버 이니셜라이저는 선언부가 아닌 정의부에 한다. 물론 C++11 이후로는 상수로 초기화하는 것은 멤버 변수에서 바로 해도 된다. 초기 cpp에서는 멤버 변수 순서대로 해야 했는데 최근에는 순서가 달라도 상관 없다. 

## **06) if문과 생성자, 소멸자**

```c++
class Player
{
pubilc: 
    Player() …
    ~ Player() …
	void Damage() …
    void Move() …

private:
	int _x;
	int _y;
	int _id;
	int _hp;
}

```c++
int main()
{
	…
    Player player;
	if(flag == 0)
	{
		
	}
}
```
```c++
int main()
{
	…
	if(flag == 0)
	{
		Player player;
	}
}
```
if 문 내부에서 선언하던 밖에서 선언하던 stack에서 자리 차지하는 정도는 동일하다. 그것까지 고려해서 stack이 잡아버리기 때문이다. 하지만 생성자, 소멸자가 있을 경우 위 아래 케이스의 생성자, 소멸자 호출 타이밍이 달라지게 된다. 만약 분기 안에서 생성했는데 분기에 들어오지 않는다면 불필요한 생성자, 소멸자 호출이 일어나지 않을 것이다. 

컴파일러의 버전에 따라서 생성자 소멸자를 inline 처리 해주는 경우도 있으니 내가 사용하는 컴파일러에서도 되는지 확인해보고 참고하는 것이 좋다.

## **07) exit과 소멸자**

```c++
if(flag == 0)
{
	Player player;
    exti(0);
}
```

exit()는 런타임 라이브러리이기 때문에 전역에 만들어진 객체의 소멸자 호출까지는 보장을 해준다. 반면 이처럼 스택에 생성된 객체의 소멸자 호출은 보장하지 않는다. 코드 영역을 확인해보면 블록이 끝나는 마지막에 소멸자 명령이 들어있기 때문이다. 소멸자에서 다른 서버에게 신호를 주거나 로깅을 하게 된다면 이러한 작업이 생략될 수 있다. 

std iterator의 경우 iterator만 container를 보는 게 아니라 container도 iterator를 보면서 개수를 container 내부에서 체인으로 관리한다. 그래서 과거에는 container를 참조하는 iterator가 아직 있는데 container를 지울 경우 소멸자에서 체킹하며 에러가 발생시켰다. 현재는 관리하는 코드는 존재하지만 에러를 발생시키지는 않는다. 이러한 경우에도 만약 container를 전역에, iterator를 지역에서 사용할 경우에 exit를 했을 때 문제가 발생할 수 있다.

## **08) 디폴트 복사 생성자**

```c++
int i2 = i1;
int i2(i1);
Sosimple sim2 = sim1;
Sosimple sim2(sim1);
```
디폴트 복사 생성자가 생긴다고 설명하는 경우도 많은데, 이는 논리적으로 그렇다는 거지 실제 코드 영역에 복사 생성자가 생긴다는 의미는 아니다. 두 경우 모두 디스어셈블리를 확인해보면 mov sim2 sim1으로 동작한다. 

## **09) friend**

```c++
class Boy
{
	friend class Girl
	// …
}
```
위와 같이 friend로 선언할 경우 해당 클래스의 private 멤버의 접근을 허용하게 된다. 정보 은닉에 반하는 선언이기 때문에 매우 제한적으로 선언되어야 한다. 네트워크 라이브러리 등을 만들 때 일반 사용자에게는 감추고 싶으나 성능적인 이유로 내가 만든 클래스에게는 허용하고 싶은 상황에서 유용하게 사용할 수 있다. 

## **10) static**

클래스 멤버 변수를 static으로 선언하면, 이름만 CTest::ctestStatic 처럼 사용해야 할 뿐이지 공용으로 사용하는 하나의 전역변수처럼 동작한다. 
 
```c++
class Boy
{
	inline static int _sss; 
}
 
int main()
{
    boy0._sss = 0;
    boy1._sss++;
    boy2._sss++;
    boy3._sss++;
    printf(“%d”, boy0::_sss);
}
```
이 경우 3이 출력된다. visual studio 17 이전에는 static 멤버 변수 역시 전역 변수이기 때문에 사용하기 전에 선언이 필요했다. 그러나 최근에는 inline으로 class 내부에서 inline으로 선언하는 것도 가능해졌다. inline과는 용도가 살짝 다르기 때문에 기존 inline의 확장된 느낌보다는 inline static 을 하나의 키워드로 보는 것이 낫다. 
 