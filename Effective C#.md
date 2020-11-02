
# Effective C# #

## C# 언어 요소 ##

### 1. 지역 변수를 선언할 때는 var를 사용하는 것이 낫다 ###

C# 언어가 익명타입을 지원하기 위해서 타입을 암시적으로 선언할 수 있는 손쉬운 방법을 제공한다.

```C#
var foo = new { gameObject.name, gameObject.transform };
Debug.Log(foo.name + foo.transform.name);
```

일부 LINQ 구문의 경우 IEnumerable<T>를 반환하지 않고 IQueryable<T>을 반환하는 경우가 있는데,
이 경우 IQueryable<T>컬렉션을 IEnumerable<T>로 형변환 하는 경우 [IQueryProvider가 제공하는 장점을 사용할 수 없게 된다.](#47-ienumerable-데이터-소스와-iqueryable-데이터-소스를-구분하라)

```C#
// 1
IEnumerable<string> query1 = from c in db.Customers
                            select c.ContactName;

var query2 = query1.Where(s => s.StartsWith(start));  // 확장 메서드 Queryable.Where이 아닌, Enumerable.Where이 호출되어 비효율적으로 동작한다.

// 2
var query1 = from c in db.Customers
            select c.ContactName;

var query2 = query1.Where(s => s.StartsWith(start));  // query1이 IQuerayable<string>으로 평가되어 Queryable.Where을 호출한다.
```

확장 메서드는 virtual로 선언할 수 없으므로 객체의 런타임 타입에 따라 다르게 동작하도록 작성할 수 없다.
또한 확장 메서드는 반드시 static으로 선언해야 하기 때문에 컴파일러는 객체의 컴파일 타임 타입에 준하여 메서드를 수행한다.
따라서 [늦은 바인딩](https://www.geeksforgeeks.org/early-and-late-binding-in-c-sharp/) 메커니즘이 적용되지 않는다.

### 2. const보다는 readonly가 좋다 ###

컴파일 타임 상수(const)는 속도면에서 런타임 상수(readonly)보다 약간 빠르지만, 유연성이 떨어진다.

const의 경우 컴파일 타임에 상수로 IL코드를 대체하고, readonly의 경우 런타임에 상수를 할당하고 참조하는 형태로 동작한다.
따라서 const는 내장 자료형, enum, null에 대해서만 사용할 수 있는 제한이 있다.

또한, const를 포함하는 어셈블리(A)를 사용한 프로젝트(P)의 경우 A의 상수만 변경하여 어셈블리를 교체한 경우, P는 교체 전 상수로 빌드하였기 때문에 [리빌드 하지 않으면 변경된 A의 상수와 다른 값을 사용](https://docs.microsoft.com/ko-kr/dotnet/csharp/whats-new/version-update-considerations#source-compatible-changes)하게 된다.

명명된 매개변수(named parameter)나 선택적 매개변수(optional parameter)도 상수 특성과 연관이 있다.
선택적 매개변수의 기본값은 const 형태로 메서드를 사용하는 호출 측에 저장된다.

그 외에도 컴파일 할 때 사용되는 상숫값을 정의할 때는 반드시 const를 사용해야 한다.

* 특성(Attribute)의 매개변수
* switch/case 문의 레이블
* enum 정의 시 사용하는 상수

이런 몇가지 예외적인 상황을 제외한다면 대부분의 경우 const보다는 readonly를 사용하는 것이 좋다.

### 3. 캐스트보다는 is, as가 좋다 ###

형변환을 수행하는 경우 캐스팅 보다는 as 연산자를 사용하는 것이 좋다. 안전함과 동시에 런타임에 더 효율적으로 동작한다. 다만, as/is 연산자를 사용하는 경우 사용자 정의 형변환은 수행되지 않는다.

형변환 과정에서 새로운 개체가 생성되는 경우는 as 연산자를 이용하여 박싱된 값 타입의 객체를 nullable 값 타입 객체로 변환하는 경우만 존재한다.

```C#
public class T
{
  private V _value;

  // T -> V로 암시적 형변환을 구현
  public static implicit operator V(T t) => t._value;
}

public static class Factory
{
  // T를 반환하는 Factory Class
  public static object GetObject() => new T();
}

var o = Factory.GetObject();
var v1 = o as V;  // 실패. as 연산자는 사용자 정의 형변환을 호출하지 않는다.
var v2 = (V)o;    // 실패. 컴파일러는 object -> V 의 사용자 형변환을 찾아보고 실패한다.
```

캐스팅은 컴파일 타임의 정보를 기반으로 동작하기 때문에, o의 런타임 정보를 가지고 T -> V 사용자 정의 형변환을 동작시킬 수 없다.

아래의 경우는 as 연산자가 동작하지 않는 경우이다.

```C#
object o = Factory.GetValue();
var i1 = o as int;   // 컴파일 오류. int는 as 형변환 실패시의 null 값을 처리할 수 없다.
var i2 = o as int?;  // 성공. nullable 타입으로 형변환하여 사용한다.
```

.NET Base Class Library(BCL)에는 시퀀스 내의 개별 요소들을 특정 타입으로 형변환하는 [Enumerable.Cast<T>()](https://docs.microsoft.com/ko-kr/dotnet/api/system.linq.enumerable.cast) 와 같은 함수가 있다. 이 함수는 IEnumerable 인터페이스만을 지원하는 컬렉션에 포함된 각각의 객체에 대해 형변환을 수행할 때 주로 사용된다. 이 함수는 as 연산자 대신 캐스트 연산을 사용하는데, as 연산자를 사용하면 형변환 하려는 타입에 제한이 생기기 때문이다. Cast<T>의 타입 매개변수에 사용자 정의 타입을 전달하는 경우 캐스트 연산으로 형변환을 수행해도 문제가 없을 지 살펴보고, 필요에 따라 사용자 정의 타입에 제약 조건을 추가할 것인지에 대해서도 검토해야 한다.
  
또한 IEnumerable<T>와 같은 제네릭 컬렉션에 대해서는 Cast<T>를 호출할 수 없다.
  
가능하면 형변환을 피하는 것이 좋지만, 불가피한 경우 사용자의 의도를 표현할 수 있는 is/as 연산자를 사용하는 것이 좋다.

### 4. string.Format()을 보간 문자열로 대체하라 ###

string.Format()의 경우 아래의 단점이 존재한다. 보간 문자열의 경우 아래 단점을 보완한다.

* 포맷 문자열과 인자 리스트를 분리하여 전달하는 구조이기 때문에 가독성이 떨어짐
* 포맷 문자열에 나타낸 인자 갯수 및 순서와 인자 리스트가 일치하지 않는 경우 런타임 에러 발생

사용자가 문자열 보간 기능을 사용하더라도 실제 C# 컴파일러는 기존 string.Format 함수를 호출하기 때문에, 인자로 값 타입을 사용하는 경우 박싱을 수행해야 한다.
자주 사용하는 코드나 루프 내에서 [박싱이 반복되지 않도록 한다.](#9-박싱과-언박싱을-최소화-하라)

```C#
Console.WriteLine($"PI : {Math.PI}");   // boxing 발생
Console.WriteLine($"PI : {Path.PI.ToString()}");    // boxing이 발생하지 않음
```

숫자의 출력 형식이나, [문화권별로 다른 문자열](#5-문화권별로-다른-문자열을-생성하려면-FormattableString을-사용하라)을 생성하는 내장된 [표준 포맷 문자열](https://docs.microsoft.com/ko-kr/dotnet/standard/base-types/standard-numeric-format-strings)의 대부분을 간 문자열에서 사용할 수 있다.

```C#
Console.WriteLine($"PI : {Math.PI.ToString("F2")}");    // 일반적인 표준 포맷 문자열 사용 형태
Console.WriteLine($"PI : {Path.PI:F2}");    // 보간 문자열에서 제공하는 표준 포맷 문자열 사용 형태
```

단, :연산자는 조건 연산자로도 사용하기 때문에, 보간 문자열 내에서 사용하는 경우 조건식에서 사용한다는 것을 알려줘야한다.

```C#
Console.WriteLine($"PI : {round ? Math.PI : Math.PI.ToString("F2")}");  // 컴파일 오류
Console.WriteLine($"PI : {(round ? Math.PI : Math.PI.ToString("F2"))}");  // ok
```

### 5. 문화권별로 다른 문자열을 생성하려면 FormattableString을 사용하라 ###

[문자열 보간 기능](#4-string.Format()을-보간-문자열로-대체하라)을 사용하여 문자열을 만드는 경우, 컴파일러는 몇가지 조건을 고려하여 string 혹은 [FormattableString](https://docs.microsoft.com/ko-kr/dotnet/api/system.formattablestring) 객체를 생성한다.

아래는 특정 문화권과 언어를 지정하여 문자열을 반환하는 방식의 예제이다.

```C#
public static string ToKorean(FormattableString text)
  => string.Format(null, System.Globalization.CurtureInfo.CreateSpecificCulture("ko-kr"),
                    text.Format, text.GetArguments());

public static string ToFrenchCanada(FormattableString text)
  => string.Format(null, System.Globalization.CurtureInfo.CreateSpecificCulture("fr-CA"),
                    text.Format, text.GetArguments());
```

변수 타입을 var로 선언한 후에 보간 문자열이 FormattableString으로 반환하기를 기대한다면, 아래 사항을 주의해야 한다.

* 문자열을 매개변수로 취하는 함수에 이 변수를 전달하는 코드를 작성하지 않는다.
* string과 FormattableString을 모두 받아들이는 오버로딩 함수에 이 변수를 전달하지 않는다.
* 반환 객체에 operator. 를 사용하는 확장 함수를 호출하지 않는다.

문자열 보간 기능은 글로벌/지역화에 필요한 거의 모든 기능을 갖추고 있고, 문화권을 고려하여 문자열을 생성하는 복잡한 내부를 잘 감추고 있다.
문화권을 임의로 지정해야 하는 경우에는 명시적으로 FormattableString 타입의 객체를 생성하도록 코드를 작성하고, 이 객체를 통해 문장려을 얻어오는 방법을 사용하는 것이 좋다.

### 6. nameof() 연산자를 적극 활용하라 ###

[nameof() 연산자](https://docs.microsoft.com/ko-kr/dotnet/csharp/language-reference/operators/nameof)는 심볼 그 자체를 해당 심볼을 포함하는 문자열로 대체해준다.
타입, 변수, interface, namespace에 대해서도 사용 가능하며, [정규화 된 이름](https://docs.microsoft.com/ko-kr/dotnet/csharp/language-reference/language-specification/basic-concepts#fully-qualified-names)도 제한없이 사용 가능하다.
정규화된 이름을 사용하더라도 항상 로컬 이름을 문자열로 반환한다. [verbatim 식별자](https://docs.microsoft.com/ko-kr/dotnet/csharp/language-reference/tokens/verbatim)를 사용하는 경우 @문자는 제외된다.

Attribute의 인자로 문자열을 전달해야 하는 경우에도 사용 가능하며, MVC Application이나 Web API의 Route 지정시에 특히 유용하다.
심볼의 이름을 수정하는 경우에도 Refactor 등으로 변경사항을 쉽게 반영할 수 있다.

가능한 한 문자화되지 않은 형태로 심볼을 유지할 수 있다면 자동화 도구를 활용할 수 있는 가능성이 높아지기 때문에, 오류 검출이 쉬워진다.

### 7. 델리게이트를 이용하여 콜백을 표현하라 ###

여러 클래스가 상호 통신을 수행해야 할 때, 클래스 간의 결합도를 낮추고 싶다면 interface보다 delegate를 사용하는 것이 좋다.
delegate는 런타임에 통지 대상을 설정할 수 있고, 다수의 클라이언트에게 통지를 보낼 수도 있다.

동일한 타입의 매개변수를 취하더라도 반환 타입이 다른 경우 서로 다른 delegate 타입으로 간주하며, 컴파일러는 이 둘 사이의 형변환을 허용하지 않는다.

LINQ는 모두 delegate 기반으로 설계되어 있으며, .NET Framework에서 매개변수로 단일 메서드를 필요로 하는 모든 경우에 lambda 표현식을 쓸 수 있도록 delegate를 사용한다. WPF나 WinForms에서 여러 Thread를 넘나들 경우 반드시 Mashaling이 필요한데, 이 경우에도 Callback이 사용된다. 이런 이유로 동일한 구조의 API를 직접 만드는 경우에도 가능한 한 이와 유사한 방식으로 작성하는 것이 좋다.

delegate는 기본적으로 multicast가 가능하다. 하지만 두가지 주의해야 할 부분이 있다.

* 예외가 발생할 경우, multicast가 중단될 수 있다.
* 마지막으로 호출한 대상 함수의 반환값이 delegate의 반환값으로 간주된다.

이런 두가지 문제를 해결하려면, 아래 예제처럼 delegate에 등록된 함수 목록을 가져와 직접 해결해야 한다.

```C#
public void foo(Func<bool> predicate)
{
  // multicast 호출 중 predicate가 false를 반환하는 경우 호출을 중단하는 형태
  foreach (Func<bool> d in predicate.GetInvocationList())  // Delegate.GetInvocationList()는 Delegate[] 를 반환한다.
  {
    if (!d())
      break;
  }
}

public int bar(Func<int, int> functions)
{
  // function의 모든 반환값을 더해 반환
  var result = 0;
  foreach (Func<int, int> d in functions)
  {
    try
    {
      result += d(1);
    }
    catch { } // 예외 발생에도 function chain의 call을 유지
  }

  return result;
}
```

delegate는 런타임에 callback을 구성하는 최고의 방법이다. delegate를 사용하면 interface를 사용하는 경우보다 callback을 사용해야 하는 클라이언트를 더욱 단순하게 구성할 수 있을 뿐 아니라 런타임에 callback 함수를 구성할 수 있다. 더해서 multicast 도 지원하기 때문에, .NET 환경에서 callback이 필요한 경우에는 반드시 delegate를 이용하기를 권장한다.

### 8. 이벤트 호출 시에는 null 조건 연산자를 사용하라 ###

event 호출시 event handler의 null 조건 연산자를 이용하여 호출하도록 권장한다.

```C#
private void EevntRaised(Action e)
{
  // case 1 : multi thread 환경에서 e()가 무효화 된 상태로 호출 가능
  if (e != null)
    e();

  // case 2 : event의 복사본을 검사 후 생성. thread-safe 하지만 코드가 복잡하다.
  var local_e = e;
  if (local_e != null)
    local_e();

  // case 3 : thread-safe하며 간결한 호출
  e?.Invoke();
}
```

### 9. 박싱과 언박싱을 최소화 하라 ###

대부분의 경우 .NET 2.0에 추가된 제네릭 클래스와 제네릭 메서드를 사용하면 [boxing/unboxing](https://docs.microsoft.com/ko-kr/dotnet/csharp/programming-guide/types/boxing-and-unboxing)을 피할 수 있다.

하지만 아래의 예제처럼 간단한 코드에서도 boxing/unboxing이 발생할 수 있다.
보간 문자열은 내부적으로 string.Format() 형태로 인자가 전달되며, 해당 함수는 param object[] 형태로 인자를 전달받기 때문에 값 형식의 인자를 사용할 경우 자동으로 boxing이 발생한다.

```C#
Console.WriteLine($"{number1} {number2} {number3}");  // 3번의 boxing이 발생
Console.WriteLine($"{number1.ToString()} {number2.ToString()} {number3.ToString()}");  // boxing/unboxing이 발생하지 않음
```

값 형식은 System.Object 타입이나 여타의 인터페이스 타입으로 변경할 수 있다. 이러한 변환 작업은 암시적으로 이뤄지며, 실제로 어떤 부분에서 이러한 변환 작업이 행해지는 지도 찾아내기 어렵다. Boxing/Unboxing 작업은 객체에 대한 복사본을 생성하곤 하는데, 이로 인해 버그가 발생할 수도 있고 값 형식을 다형적으로 처리하는 과정에서 성능을 느리게 만든다.

### 10. 베이스 클래스가 업그레이드된 경우에만 new 한정자를 사용하라 ###

```C#
public class Base
{
  public void Function() => Console.WriteLine("Base");
}

public class Derived : Base
{
  // Function을 재정의
  public new void Function() => Console.WriteLine("Derived");
}
```

위와 같이 비 가상 메서드를 재정의 한 경우, 동일한 인스턴스라도 어떻게 참조하는가에 따라 호출 결과가 달라진다.
new 한정자를 활용해도 좋은 경우는 베이스 클래스(A)에서 이미 사용하고 있는 메서드를 재정의하여 완전히 새로운 베이스 클래스(B)를 만들어야 하는 경우 정도다.

아래와 같은 형태로 사용하던 중

```C#
public class A
{
  public void Function() { ... }
}

public class B : A
{
  public void MyFunction() { ... }
}
```

A클래스에 MyFunction() 이라는 함수가 구현되어 namespace에서 충돌하는 경우, 1) B클래스의 MyFunction() 이름을 변경하거나, 2) B클래스에서 MyFunction을 new 한정자로 재정의 할 수 있다.

```C#
public class B : A
{
  public new void MyFunction() => base.MyFunction();
}
```

하지만, 이 경우에도 MyFunction은 참조 방식에 따라 다른 호출을 한다는 상황은 변함 없으므로 신중히 고려해야 한다.
그 외의 경우라면 절대로 new 한정자를 사용해서는 안된다.

## .NET 리소스 관리 ##

### 11. .NET 리소스 관리에 대한 이해 ###

[Garbage Collector(GC)](https://docs.microsoft.com/ko-kr/dotnet/standard/garbage-collection/fundamentals)는 Managed Memory를 관장하며 메모리 릭, [댕글링 포인터](https://ko.wikipedia.org/wiki/%ED%97%88%EC%83%81_%ED%8F%AC%EC%9D%B8%ED%84%B0), 초기화되지 않은 포인터, 여타의 메모리 관리를 자동화 해준다.

이와는 반대로 DB Connection, GDI+ 객체, COM객체, 시스템 객체등은 개발자가 직접 관리해야 한다. Event Handler나 delegate등도 제대로 관리하지 않으면 참조하고 있는 객체가 불필요하게 오래 메모리에 남게 된다. [결과를 반환하는 Query등도 잘못 사용](#41-값비싼-리소스를-캡쳐하지-말라)하면 예상보다 더 오랫동안 메모리를 점유하게 된다.

GC는 COM의 경우처럼 개별 객체가 스스로 자신의 참조 여부나 횟수 등을 관리하도록 하는 방식이 아니라, 응용 프로그램의 최상위 객체로부터 개별 객체까지의 도달 가능 여부를 확인하도록 설계했다. 도달 가능한 객체를 살아있는 객체로 판단하고, 도달 불가능한 객체를 Garbage로 간주한다. 이렇게 분류한 객체들을 Managed Heap의 Compact 과정에서 메모리 위치를 조정하여 단편화된 메모리 중 살아있는 객체들을 하나의 큰 메모리 덩어리로 합친다.

비관리 리소스의 경우, 유저가 관리하기 손쉽도록 finalizer와 IDisposable 인터페이스를 제공한다. finalizer는 비관리 리소스의 해제 작업이 반드시 수행될 수 있도로 도와주는 방어적인 매커니즘이지만 단점이 많기 때문에 IDisposable 인터페이스를 통해서 비관리 리소스를 해제하기를 권장한다.

finalizer를 가지고 있는 객체는 finalizer를 호출해야 하기 때문에 GC가 바로 메모리를 해제 하지 못한다. 또한, GC가 동작하는 Thread에서 finalizer를 호출할 수 없기 때문에 해당 객체를 [finalizer Queue](https://docs.microsoft.com/ko-kr/dotnet/api/system.gc.waitforpendingfinalizers)에 삽입하는 사전 준비 작업만 진행한다. 다음 GC 수집 절차가 수행되면, finalizer Thread에서 finalizer Queue에 존재하는 finalizer 포함 객체들의 finalizer를 호출한 후 메모리를 해제한다.

위 설명처럼 finalizer를 가지고 있는 객체는 Garbage로 간주된 이후에도 오랜 시간 메모리를 점유하게 되며, 사용자가 구현한 finalizer는 긴 시간이 지난 후에 GC에 의해 호출된다. 언제 호출되는지 사용자는 알 수 없다. 특정 타이밍에 명시적으로 리소스를 해제해야 하는 경우에는 사용할 수 없다.

GC는 수집 과정을 최적화 하기 위해 [세대(Generation)](https://docs.microsoft.com/ko-kr/dotnet/standard/garbage-collection/fundamentals#generations)라는 개념을 사용한다.
최종 GC 수집 이후 생성된 객체는 0세대 객체이다. 이후에 GC 수집이 발생하여 0세대 객체중 살아있는 객체를 1세대 객체가 된다. 2번 혹은 그 이상의 GC 수집 절차에 살아남은 1세대 객체들은 2세대 객체로 구분한다. 0세대 객체는 높은 확률로 Gabage가 되기 때문에 GC 수집과정 매번 0세대 객체를 대상으로 Garbage 수집을 진행한다. 대략 10번에 한번꼴로 1세대 객체에 대한 수집을 수행하고, 100번에 한번 꼴로 2세대 객체를 포함한 모든 세대의 객체를 대상으로 Garbage 수집을 진행한다.

finalizer를 포함하는 객체의 경우 0세대에서 메모리가 해제되지 않으므로 1세대 객체가 되고, 1세대 객체 수집이 수행되지 않는 중에 2세대 객체가 될 수도 있다. 이 경우 해당 객체는 오랜 시간 메모리에 잔류하여 해제되지 않게 된다. [Dispose 패턴을 구현](#17-표준-Dispose-패턴을-구현하라)하여 finalizer 리소스 해제를 효과적으로 대체할 수 있다.

### 12. 할당 구문보다 멤버 초기화 구문이 좋다 ###

클래스의 멤버 변수들의 값을 초기화 할 때, 생성자 할당 구문보다 멤버 초기화 구문을 사용하는 것을 권장한다.

* 여러개의 생성자를 사용하는 경우 초기화 코드를 누락하는 것을 방지
* 변수 선언과 객체 생성이 동시에 작성된 형태가 자연스러움
* 생성자를 갖지 않는 경우에도 초기화 동작
* 생성자 이전에 초기화 구문이 수행되므로, 상속받은 클래스의 생성자에서 베이스 클래스의 생성자가 호출되기 전에도 안전하게 접근 가능

예외적으로 아래의 3가지 경우는 멤버 초기화를 안하는 것이 좋다.

첫째는 객체를 0이나 null로 초기화 하는 경우이다. 시스템 초기화 루틴은 저수준에서 직접 CPU명령을 수행하여 메모리 블록을 0으로 설정하기 때문에 중복 동작을 하는 코드를 생성하게 된다. 이 경우 [박싱/언박싱된 변수](#9-박싱과-언박싱을-최소화-하라) 모두에 대해서 0으로 초기화 하는 과정([IL initobj](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.initobj))이 수행되어 성능 저하가 있다.

```C#
public struct Value { ... }

class foo
{
  private Value v1; // 메모리 블록 0으로 초기화
  private Value v2 = new Value(); // IL initobj
}
```

둘째는 동일한 객체를 반복해서 초기화 하는 경우다. 이 경우 멤버 초기화 구문에 의해 생성된 객체는 0세대에 즉각 가비지가 된다.
멤버 초기화 구문은 모든 생성자가 동일한 방법으로 멤버 변수를 초기화 하는 경우에만 사용해야 한다.
암시적인 속성을 사용하는 경우에도 [중복 초기화](#14-초기화-코드가-중복되는-것을-최소화하라)가 발생할 수 있다.

```C#
public class foo
{
  private List<string> list = new List<string>();
  public foo()
  {

  }
  
  pubilc foo(int size)
  {
    list = new List<string>(size);
  }
}
```

셋째는 예외처리가 반드시 필요한 경우이다. 멤버 초기화 구문은 try-catch 를 사용할 수 없기 때문에 초기화 과정에서 예외 발생시 예외가 외부로 전파된다.
이 경우 생성자 내부로 초기화 코드를 옮기고 [예외 처리 코드를 적절히 구현](#47-사용자-지정-예외-클래스를-완벽하게-작성하라)해야 한다.

### 14. 초기화 코드가 중복되는 것을 최소화하라 ###

생성자를 이용하여 멤버 변수의 값을 초기화 하는 경우, 다른 생성자를 호출하여 초기화 과정을 위임하는 방식([생성자 체인 기법](https://www.codeproject.com/Articles/271582/Constructor-Chaining-in-Csharp-2))으로 초기화 하는 것을 권장한다.

변수를 초기화하는 멤버함수를 구현하는 경우, 읽기 전용 변수등의 초기화는 해당 함수에서 처리할 수 없다.

생성자 매개변수에 기본값을 지정하여 사용하는 방식으로 호출자가 선택적으로 생성자 매개변수의 값을 전달할 수 있도록 구현할 수 있으나, 몇가지 제약사항이 있다.

* 기본값을 갖는 매개변수를 취하는 생성자는 제네릭의 new() 조건을 만족하지 못한다.
* Reflection을 이용하여 인스턴스를 생성하는 경우 기본 생성자를 제공해야 한다.
* 매개변수의 이름이 변경되는 경우, 매개변수의 이름이 어셈블리 외부로 노출되어 동작하기 때문에 [소스 호환 가능 변경](https://docs.microsoft.com/ko-kr/dotnet/csharp/whats-new/version-update-considerations#source-compatible-changes) 이다.

```C#
public class foo
{
  private readonly List<int> container;
  private int value;
  private string name;

  public foo() : this(0, string.Empty)
  {
  }

  public foo(int v = 0, string n = "")
  {
    value = v;
    name = n;
    container = new List<int>();
  }
}
```

### 17. 표준 Dispose 패턴을 구현하라 ###

[표준 Dispose 패턴](https://docs.microsoft.com/ko-kr/dotnet/standard/garbage-collection/implementing-dispose#implement-the-dispose-pattern)은 GC 수집기와 연게되어 동작하며, 불가피한 경우에만 finalizer를 호출하도록 하여 성능에 미치는 부정적인 영향을 최소화 한다.

상속 계통상 최상위의 베이스 클래스는 다음과 같은 작업을 수행해야 한다.

* 비관리 리소스를 정리하기 위해서 interface IDisposable을 구현해야 한다.
* 멤버 필드로 비관리 리소스를 포함하는 경우에 한해 방어적으로 동작할 수 있도록 finalizer를 추가해야 한다.
* Dispose와 finalizer(존재하는 경우)는 파생 클래스가 고유의 리소스 정리 작업이 필요한 경우 이 가상 메서드를 재정의할 수 있도록 실제 리소스 정리 작업을 수행하는 다른 가상 메서드에 작업을 위임하도록 작성되어야 한다.

파생 클래스는 다음 작업을 수행해야 한다.

* 파생 클래스가 고유의 리소스 정리 작업을 수행해야 한다면 베이스 클래스에서 정의한 가상 메서드를 재정의 한다.
* 멤버 필드로 비관리 리소스를 포함하는 경우에만 finalizer를 추가해야 한다.
* 베이스 클래스에서 정의하고 있는 가상 함수를 반드시 재호출해야 한다.

비관리 리소스를 포함하는 클래스는 반드시 finalizer를 구현해야 한다. Dispose()가 호출되지 않는다면 해당 리소스는 누수되기 때문에, 방어적으로 동작하기 위해 finalizer를 구현해야 한다.

[interface IDisposable](https://docs.microsoft.com/ko-kr/dotnet/api/system.idisposable)의 Dispose() 메서드는 다음 4가지 작업을 반드시 수행해야 한다.

* 모든 비관리 리소스를 정리한다.
* 모든 관리 리소스를 정리한다.
* 객체가 이미 정리되었음을 나타내기 위한 상태 플래그를 설정한다. 이미 정리된 객체에 대하여 추가로 정리 작업이 요청될 경우 이 플래그가 올라가있다면 ObjectDisposed 예외를 발생시킨다.
* GC.SuppressFinalize(this)를 호출하여 finalizer 호출을 회피한다.

finalizer와 Dispose()에서는 동일한 역할을 수행하므로, 코드 중복이 일어날 수 있기 때문에 표준 Dispose 패턴에서는 3번째 Helper 메서드를 사용한다.
해당 함수의 원형은 아래와 같다.

```C#
protected virtual void Dispose(bool isDisposing);
```

이 가상함수를 구현하여 관리/비관리 리소스 해제를 구현하면 중복 코드 없이 finalizer와 Dispose() 양쪽에서 사용할 수 있다. 또한 가상 함수이므로 파생 클래스에서 이 메서드를 재정의하여 자신이 소유한 리소스를 정리하는 코드를 작성할 수 있다. 이 경우, 코드의 마지막 부분에서는 반드시 베이스 클래스에서 정의하고 있는 Dispose(bool) 함수를 호출해야 한다.

이 Dispose(bool) 함수를 호출할 때에는 **관리/비관리 리소스 모두를 제거하려면 isDisposing으로 true를 전달하고, 비관리 리소스만 정리하려면 false만 전달** 해야 한다.
아래는 표준 Dispose 패턴을 구현한 Base/Derived 클래스 예제이다.

```C#
public class Base : IDisposable
{
  private bool disposed;  // dispose flag

  // IDisposable 구현
  // Dispose()가 호출되었다면 finalizer를 회피한다.
  public void Dispose()
  {
    Dispose(true);
    GC.SuppressFinalize(this);
  }

  // 가상 Dispose 함수
  protected virtual void Dispose(bool isDisposing)
  {
    // 한번만 수행하도록 보장한다.
    if (disposed)
      return;

    if (isDisposing)
    {
      // 관리 리소스 정리
    }

    // 비관리 리소스 정리
    disposed = true;
  }

  public void foo()
  {
    if (disposed)
      throw new ObjectDisposedException("이미 해제된 객체의 함수를 호출했습니다.");
  }
}
```

```C#
public class Derived : Base
{
  private bool disposed_derived;  // 자신만의 disposed flag를 갖는다.

  protected override void Dispose(bool isDisposing)
  {
    // 한번만 수행하도록 보장한다.
    if (disposed_derived)
      return;

    if (isDisposing)
    {
      // 관리 리소스 정리
    }

    // 비관리 리소스 정리

    // 베이스 클래스의 리소스 정리
    base.Dispose(isDisposing);

    disposed_derived = true;
  }
}
```

베이스/파생 클래스의 일부만이 정리된 경우에 혹시 발생할지 모르는 문제를 피하기 위해 각자의 disposed flag를 사용하여 구현한다.
Dispose() 메서드는 여러번 호출되더라도 동일하게 동작하도록 구현해야 한다. 이미 정리된 객체의 멤버 메서드를 호출한 경우 ObjectDisposedException 예외를 발생시키는 것도 표준 Dispose 패턴의 규칙이다.

제거/정리 작업에 있어서 가장 핵심적이고 중요한 지침중 하나는 Dispose(bool) 메서드 내에서는 리소스 정리 작업만을 수행하라는 것이다. Dispose()나 finalizer에서 다른 작업을 수행하게 되면 객체의 생명주기와 관련된 심각한 문제를 일으킬 수 있다. finalizer를 가진 객체는 최종적으로 정리가 완료되기 이전에 객체를 다시 도달 가능 상태로 만들어버리면 객체는 해제되지 않고, 비정상적인 상태로 남게 된다.

* GC는 해당 객체의 finalizer를 호출했으므로 더이상 finalizer를 호출할 필요가 없다고 간주한다. 따라서 되살아난 객체가 다시 제거되는 과정에서 finalizer를 호출할 방법이 없다.
* finalizer 과정이 완료된 객체는 명백히 가비지로 간주되어, 모든 관리 멤버 필드등이 메모리에서 해제된다.

## LINQ 활용 ##

### 37. 쿼리를 사용할 때는 즉시 평가보다 지연 평가가 낫다 ###

쿼리를 정의하는 작업은 작업 수행 절차를 정의하는 것이므로, 쿼리의 결과를 순회하는 경우에 결과가 생성된다. 이를 지연 평가(Lazy Evaluation)라고 한다.
일반 변수를 사용하는 것 처럼 즉각적으로 값을 게산하는 방식은 즉시 평가(Eager Evaluation 혹은 Strict Evaluation)라고 한다.

쿼리 표현식은 지연평가를 수행하기 때문에 이론적으로 무한한 시퀀스를 표현하는 것이 가능하다. 첫번째 값부터 시작하여 적절한 값을 만난 경우 작업을 중단하도록 코드를 작성하기만 하면 된다.
하지만 일부 쿼리 표현식의 경우 결과를 얻기 위해서 반드시 전체 시퀀스가 준비되어야 하는 경우도 있다.

```C#
static IEnumerable<int> AllNumbers()
{
    var number = 0;
    while (number < int.MaxValue)
    {
        // int.MaxValue까지 1씩 증가하는 시퀀스 생성
        yield return number++;
    }
}

// 1. 앞에서 10개의 값만 획득. 빠르게 동작한다.
var answers = from number in AllNumbers()
                select number;
var take10 = answers.Take(10);

// 2. 모든 컬렉션 값에 접근.
var answers = from number in AllNumbers()
                where number < 10   // where 구문은 전체 시퀀스를 필요로 한다.
                select number;
```

where, orderby, min, max 등의 구문은 동작상 전체 시퀀스를 필요로 한다. 이 경우
첫째, 시퀀스가 무한정 지속될 가능성이 있다면 위와 같은 전체 시퀀스를 필요로 하는 구문을 사용할 수 없다.
둘째, 시퀀스가 무한정 지속되지 않는 경우에도, 시퀀스를 필터링 하는 쿼리 메서드는 다른 쿼리보다 먼저 수행하는 것이 좋다.

```C#
var slowQuery = from p in products
                orderby p.UnitsInStock descending
                where p.UnitsInStock > 100
                select p;   // 정렬 후 필터링. 느리다.

var fastQuery = from p in products
                where p.UnitsInStock > 100
                orderby p.UnitsInStock descending
                select p;   // 필터링 후 정렬. 빠르다.
```

즉시 평가가 필요한 경우, ToList()나 ToArray()를 사용하여 컬렉션의 스냅샷을 얻을 수 있다.
하지만 즉시 평가가 필요한 경우가 아니라면 대체로 지연 평가를 사용하는 편이 훨씬 낫다.

### 41. 값비싼 리소스를 캡쳐하지 말라 ###

### 42. IEnumerable<T> 데이터 소스와 IQueryable<T> 데이터 소스를 구분하라 ###

IQueryable<T>와 IEnumerable<T>는 거의 동일한 API 정의를 가지지만, 항상 이 둘을 서로 대체하여 사용할 수 있는 것은 아니다.
실제로 동작 방식도 다르고 성능도 매우 크게 차이난다.

```C#
// 1.
var q = from c in dbContext.Customers
        where c.City == "London"
        select c;   // IQueryable<T> 반환

var finalAnswer = from c in q
                    orderby c.Name
                    select c;   // q의 where절과 finalAnswer의 order절이 결합된 단일 T-SQL구문을 만들어서 한차례 DB 호출을 수행한다.

// 2.
var q = (from c in dbContext.Customers
        where c.City == "London"
        select c).AsEnumerable();   // IEnumerable<T> 반환

var finalAnswer = from c in q
                    orderby c.Name
                    select c;   // City가 London인 모든 Customer를 가져와 로컬 컴퓨터에서 Name 필드 정렬을 수행한다.
```

Enumerable<T> 확장 메서드는 쿼리식 내의 람다 표현식과 함수 매개변수를 나타내기 위해서 delegate를 사용한다.
Enumerable<T>내의 모든 메서드는 로컬 머신에서 수행한다. 이로 인해 모든 데이터를 로컬 머신 메모리로 가져와야 하며, 상대적으로 더 많은 데이터를 가져와야 한다.

반면 Queryable<T>는 동일한 함수라 하더라도 [표현식 트리](https://docs.microsoft.com/ko-kr/dotnet/csharp/programming-guide/concepts/expression-trees/)를 이용하여 이를 처리한다.
Queryable<T>는 표현식 트리를 분석한 후, 분석한 로직을 IQueryProvider에 적합한 형태로 변경한 다음, 이를 데이터가 실제 위치하고 있는 머신(ex : Database, ..)에서 수행한다.

하지만 Queryable<T>가 구현하고 있는 IQueryable<T>를 이용하면 코드로 표현할 수 있는 쿼리 표현식이 IEnumerable<T>에 비해 상대적으로 제한적이다.

[IQueryable<T> 제공자는 각각의 메서드를 분석하지 않고, .NET Framework에서 구현한 연산자와 메서드 집합에 대해서만 동작한다.](#37-쿼리를-사용할-때는-즉시-평가보다-지연-평가가-낫다)
만약 쿼리 표현식이 다른 메서드를 호출하는 부분을 포함하고 있다면 Enumerable 구현체를 사용하도록 쿼리를 강제로 변경해야 한다.

```C#
private bool isValidProduct(Product p) => p.ProductName.LastIndexOf('c') == 0;

var q1 = from p in dbContext.Products.AsEnumerable()
            where isValidProduct(p)
            select p;   // 제대로 동작한다.
                        // Products의 전체 데이터를 가져와 로컬 머신에서 Linq to Objects를 이용하여 where절을 수행한다.

var q2 = from p in dbContext.Products
            where isValidProduct(p)
            select p;   // 컬렉션을 순회할 때 예외를 발생한다.
                        // LINQ to SQL은 쿼리를 내부적으로 T-SQL로 변환하는 IQueryProvider 구현체를 포함하고 있는데,
                        // 이를 통해 변환된 T-SQL은 원격지의 데이터베이스 엔진에 전달되어 SQL 구문을 수행한다.
```

IEnumerable<T>과 IQueryable<T>은 거의 동일한 기능을 가지고 있지만, 개별 인터페이스의 구현상의 차이가 명확하기 때문에 만약 데이터 소스가 IQueryable<T>을 구현한다면 반드시 이를 사용해야 한다.

아래와 같이 IEnumerable<T>와 IQueryable<T>를 이용한 쿼리를 모두 지원해야 하는 경우,

```C#
public static IEnumerable<Product> ValidProducts(this IEnumerable<Product> products) =>
    from p in products
    where p.ProductName.LastIndexOf('C') == 0
    select p;

public static IQueryble<Product> ValidProducts(this IQueryable<Product> products) =>
    from p in products
    where p.ProductName.LastIndexOf('C') == 0
    select p;
```

AsQueryable()을 사용하여 중복을 제거할 수 있다.

```C#
public static IEnumerable<Product> ValidProducts(this IEnumerable<Product> products) =>
    from p in products.AsQueryable()    // 시퀀스의 런타임 타입이 IEnumerable인 경우
                                        // LINQ to Objects를 사용하여 IQueryable 타입으로 래핑된 객체를 반환한다.
    where p.ProductName.LastIndexOf('C') == 0
    select p;
```

위의 `string.LastIndexOf()`는 LINQ to SQL 라이브러리를 통해 정상 동작하는 메서드중 하나이지만, 각각의 제공자들은 각기 제공하는 기능이 다르므로 IQueryProvider 구현체가 이 메서드를 구현하고 있다고 가정해서는 안된다.

## 예외 처리 ##

### 47. 사용자 지정 예외 클래스를 완벽하게 작성하라 ###
