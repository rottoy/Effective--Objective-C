
## 정의
<br/>

식이 끝난 후 계속 존재하는 값은 좌측값, 식이 끝나면 존재하지 않는 임시값은 우측값이다.

LValue : Callable

RValue : Temporary

우측값과 좌측값을 판단하는 예시이다.

```cpp
int main()
{
    int nCount = 0;
    nCount; // LValue
    0; //RValuer
 
    PLAYER Player;
    Player; // LValue
    PLAYER(); //Rvalue
 
    foo(); // Rvalue
		int i = 42;
		i = 43;            // ok, i 는 좌측값
		int* p = &i;       //&i 를 쓸 수 있다.
		int& foo();        // int& 을 리턴하는 함수
		foo() = 42;        // ok, foo() 는 좌측값
		int* p1 = &foo();  // ok, &foo() 를 할 수 있다.

		// 우측값들
		//
		int foobar();  // int 를 리턴하는 함수
		int j = 0;
		j = foobar();         // ok. foobar() 는 우측값이다
		int* p2 = &foobar();  // error. 우측값의 주소는 참조할 수 없다.
		j = 42;               // 42 는 우측값이다.
}

```
<br/>

좌측값과 우측값 참조


```cpp
int a = 10;
int& refA = a; // 좌측값 참조

int nCount;
int& LrefValue1 = nCount;  // Ok
int& LrefValue2 = 10t; // error
 
int&& RrefValue1 = 10; // ok
int&& RrefValue2 = nCount; //error

```

<br/>
잠깐! 그런데 쓸모없어 보이는 우측값 참조는 왜 쓰는 것일까?
<br/><br/>

## 우측값 참조의 탄생 배경

- 우측값 참조는 C+11에서 만들어진 새로운 문법이다. 이 문법이 추가되게 된 배경은 불필요한 우측값의 Copy Semantics를 Move Semantics로 바꾸려고 하기 때문이다.

Copy Semantics

이를 설명하기 위해 다음과 같은 코드를 보자.

```cpp
int main(){

	std::string a ="abc";

  std::vector<std::string> vec;
	vec.push_back(a);
	vec.push_back("def");
}
```

<br/>
4번째 줄을 보면벡터 클래스 벡터에 a를 넣게 되는데, 이 때 a는 Stack영역에 존재하는 좌측값이다. 

프로그래머는 벡터에 a가 넣어져도 이후에 a를 수정할 수 있게끔 해야하므로 벡터의 원소와 string a는 독립적으로 존재해야한다. 따라서 벡터에 a를 푸시할 때 Copy Semantics가 적용된다. 말 그대로 원소가 복사되어 벡터에 푸시되는 것이다.

반면, 5번째 줄을 보면 문장이 실행되면 곧 사라질 우측값 string `"def"`가 존재한다.

우측값을 좌측값과 동일하게 Copy Semantics를 적용하면 바람직할까?

정답은 `아니다` 이다. 우측값은 이후에 사라질 운명이므로, 누군가 더 이상 참조할 일이 없는 값이다. 따라서 굳이 Copy를 할 필요가 없다. Copy를 하지 않고, 현재 우측값 `"def"`의 주소를 바로 벡터의 원소에 참조시켜 주면 해결된다. 이를 Move Semantics라 하는데, 이를 위해서는 우측값의 주소를 참조해야 하므로  `C++11`  에서 새로 추가된 **우측값 참조 문법**이 새로 필요하다.


실제로 `vec.push_back("def")`  구문을 정의 이동해서 관찰하면 다음과 같다.

```cpp
void push_back(_Ty&& _Val)
		{	// insert by moving into element at end, provide strong guarantee
		emplace_back(_STD move(_Val));
		}
```
<br/><br/>

## 인자에 move라는 함수가 있는데 이는 다음 설명을 통해 알아보자.

좌측값도 Move Semantics 적용 가능한가요?

어떤 프로그래머는 나누어진 두 벡터를 단순히 합치고 싶다. 하지만 선언된 벡터는 기본적으로 둘 다 좌측값이고, Copy Semantics가 불가피하다. 

이럴 때 Move Semantics를 적용시킬 수 있게 도와주는 라이브러리 함수가 새로 생겼다.

`std::move()` 를 사용하자.


```cpp
template <typename T>
void Swap (T& a, T& b)
{
  T tmp = a;
  a = b;
  b = tmp;
}

```

좌측값 인자 2개를 통한 swap 동작이다. 

3줄 모두 operator= 에서 copy가 발생하므로 copy semantics가 적용된다. 이는 아까도 언급했듯이 비효율적이므로, 좌측값을 우측값으로 바꿔주는 `move()` 함수를 통해 move smenatics로 바꿀 수 있다.<br/>

```cpp
template <typename T>
void Swap (T& a, T& b)
{
  T tmp = std::move(a);
  a = std::move(b);    
  b = std::move(tmp);
}

```

복사가 발생하지 않으므로 비용이 적어진다.

이처럼 move semantics는 상황에 따라 작게는 특정 operation, 크게는 프로그램 전체의 성능을 개선시키는 데 도움을 줄 수 있다.

<br/><br/>

## 주의! 우측값 참조는 우측값이 아니다

다음과 같은 식이 있다.

```cpp
int& r1 = n;
int&& r2 = 10;
 
foo(r1); //< (1)
foo(r2); //< ??
```
<br/>

각각 우측값 인자, 좌측값 인자로 오버로딩된 함수 foo 1개씩 있다고 가정하자. 

이럴때 r2가 들어간 foo는 어떤 함수를 실행할까?

정답은 r1과 같은 함수(인자가 좌측값)를 실행한다. 이유는 우측값 참조는 우측값을 참조할수는 있지만, 우측값 참조 자체는 좌측값이라는 사실이다.
