---
layout: post
title: Effective C++ | 3장
subtitle: 자원 관리
categories: Effective_C++
tags: [Effective_C++, C++]
---
## 들어가며

C++에서 자원은 메모리뿐만 아니라 파일 서술자, 뮤텍스 잠금, 그래픽 유저 인터페이스와 같이 수많은 자원이 존재한다. 이는 다 쓰고 난 후에는 __"생성자/소멸자/객체 복사 함수"를__ 사용하여 해제한다.

## 항목 13 : 자원 관리에는 객체가 그만

자원 해제는 사용자가 직접 delete를 블록내에서 사용하여 객체의 자원을 해제할 수 있는데 이는 문제를 야기한다. 만약 delete가 호출되기 전에 retrun된다거나 사용하지는 않지만 goto문을 사용하여 건너뛰게 되는 경우이다.

```cpp
class Investment{...};
Investment* createInvestment();
// tr1::shared_ptr<Investment> CreateInvestment();   -> 포인터의 반환은 문제가 발생할 수 있음(delete 호출을 호출자에서 해야하는데 잊을 수도)
void f(){
  Investment *pInv = CreateInvestment();    
  ...
  delete pInv;    //혹시나 호출 안될 수도 있다.
}
```

그렇기 때문에  자원을 객체에 넣고 소멸자가 자원 해제하는 방법을 사용하여 해결하며, 이 방법에는 2가지가 존재한다. 하지만 이 스마트 포인터의 소멸자에서는 delete[]가 아닌 delete를 사용하기 때문에 동적으로 할당한 배열에 대해 사용할 수 없다. (new를 사용한 모든 것)


> 1. 스마트 포인터 : auto_ptr
  
  포인터와 비슷한 스마트 포인터로 가리키고 있는 대상에 대해 소멸자가 자동으로 delete를 불러주도록 설계되어있다. 

  ```cpp
  void f(){
    auto_ptr<Investment> pInv(CreateInvestment());  //팩토리 함수 호출
  } //소멸자로 자동 소멸
  ```

  이때 자동으로 delete하기 때문에 동일한 객체를 가르키는 auto_ptr의 개수가 둘 이상이라면 안되는데, 이를 막기 위해서 auto_ptr은 객체를 복사하면 원본 객체는 null로 만든다. 자원에 대한 유일한 소유권을 갖는다고 가정할 수 있다. 또한 중요한 2가지 특징이 존재한다.

  1. 자원을 획득한 후에 자원 관리 객체에게 넘긴다.
      - CreateInvestment()로 인해 만들어진 자원은 auto_ptr 초기화에 사용되며, 이를 __자원 획득 즉 초기화(RAII)라고__ 부른다.
      - 즉 자원의 획득과 자원 관리 객체의 초기화가 동시에 이루어진다.
  2. 자원 관리 객체는 소멸자로 자원 해제한다.
      - 블록을 어떻게 떠나던 자원 해제된다.

> 2. 참조 카운팅 스마트 포인터 : shared_ptr

  auto_ptr의 대안으로 사용하는 참조 카운팅 방식 스마트 포인터(RCSP)로 특정 자원을 가르키는 외부 객체의 개수를 유지하고 있다가 그 개수가 0이 되면 해당 자원을 가르키는 자원을 자동으로 삭제하는 스마트 포인터이다. 동작은 가비지 컬렉션과 유사하지만 참조가 고리를 이루는 경우에는 없앨 수 없다.

  ```cpp
  void f(){
    tr1::shared_ptr<Investment> pInv(createInvestment()); 
  } //소멸자로 자동 소멸
  ```

  auto_ptr과 유사하지만 복사가 훨씬 자연스러워졌으며, auto_ptr을 사용할 수 없는 STL 컨테이너등의 환경에 쓸 수 있다. 
 
### 잊지 말자

  - 자원 누출을 막기 위해, 생성자 안에서 자원을 획득하고 소멸자에서 그것을 해제하는 RAII객체를 사용하자.
    - 자원을 관리하는 객체를 써서 자원을 관리해라. 직접 호출하지 말고
  - RAII 클래스는 tr1::shared_ptr과 auto_ptr이다. 이 중 tr1::shared_ptr이 복사 시 동작이 직관적이기에 사용하기 좋다.

## 항목 14 : 자원 관리 클래스의 복사 동작에 고찰하자.

컴팡이러가 생성한 동작이 원하는 동작이 아니라면 스스로 자원 관리 클래스를 만들어야한다. 이때 복사 동작을 구현하는 것에 대해서 얘기하려고 한다. 먼저 아래는 Mutex타입을 조작하는 C API를 사용한다고 가정하자. Lock을 사용할때는 RAII의 방식에 맞게 사용하면 문제가 없지만, __복사를 하는 경우 구현에 있어 다양한 선택지가 존재한다.__
  
```c++
void lock(Mutex *pm);   //pm 잠금
void unlock(Mutex *pm); //pm 해제

class Lock{
public:
  explicit Lock(Mutex *pm) : mutexPtr(pm) {lock(mutexPtr);}
  ~ Lock() {unlock(mutexPtr);}
private:
  Mutex *mutePtr;
}

//Lock 사용 시 RAII방식에 맞게 사용
Mutex m;
lock m1(&m);

//문제는 복사
Lock m11(&m);       //m을 잠금
Lock m12(m11);      //m11을 m12로 복사 -> 이때의 선택지가 4가지 존재
```

> 1. 복사를 금지

  RAII 객체가 복사되도록 허용하는 것 자체가 말이 안되는 경우가 존재하는데 위의 예시 또한 이 부류에 속한다. __'사본'이라는__ 게 의미가 없기 때문이다. 그렇기 때문에 복사 함수를 private 멤버로 만들어서 복사를 막기만 하면 된다.
  
  ```cpp
  class Lock: private Uncopyable{}
  ```

> 2. 관리하고 있는 자원에 대해 참조 카운팅을 수행

  자원을 사용하고 있는 마지막 객체가 소멸될 때까지 자원을 해제하지 않는게 바랍직한 경우에는 객체의 개수에 대한 카운트를 증사시키는 식으로 복사 함수를 구현해야한다. 이 방식이 tr1::shared_ptr이다. 

  하지만 tr1::shared_ptr을 데이터 멤버로 넣기만 하면 안된다. 이는 참조 카운트가 0이 될 때 삭제하기 대문에 원하는 잠금을 해제하려는 의도와는 다르다. 다행이도 delete는 지정을 허용하기 때문에 __삭제자를 tr1::shared_ptr 생성자의 두 번째 매개변수(unlock)로 선택적으로 넣어주면 된다. 소멸자는 선언하지 않아도 자동으로 호출된다.__

  ```cpp
  class Lock{
  public:
    explicit Lock(Mutex *pm) : mutexPtr(pm,unlock) {lock(mutexPtr.get());}
    ~ Lock() {unlock(mutexPtr);}
  private:
    tr1::shared_ptr<Mutex> mutePtr;
  }
  ```

> 3. 관리하고 있는 자원을 복사

  자원을 원하는 대로 복사하는 경우가 존재할 수 있는데, 객체가 둘러싸고 있는 자원까지 복사되어야 한다. 즉 '깊은 복사'를 수행하여 string과 같은 경우 새로운 힙 메모리를 할당해준다.

> 4. 관리하고 있는 자원의 소유권을 옮김

  흔한 경우는 아니지만 RAII객체는 딱 하나만 존재하도록 하기 위해서 RAII 객체가 복사될 때 자원의 소유권을 사본 쪽으로 옮기는 경우이다. auto_ptr의 복사 동작과 동일하며 기존 RAII객체에는 null이 할당된다.

### 잊지 말자

  - RAII 객체의 복사는 그 객체가 관리하는 자원의 복사 문제를 안고 가기 때문에, 그 자원을 어떻게 복사하느냐에 따라 RAII 객체의 복사 동작이 결정된다.
  - RAII 클래스에 구현하는 일반적인 복사 동작은 복사를 금지하거나 참조 카운팅을 해주는 선에서 마무리하는 것이다. 이 외에도 방법은 있다.

## 출처
  ※ Effective C++ 책의 내용을 개인적으로 요악한 것입니다.
  책의 저작권 및 각종 권한은 출판사, 지은이/옮긴이에게 있음을 알립니다.
  - 지음 : 스콧 마이어스
  - 옮김 : 곽용재
  - 출판 : Addison Wesley