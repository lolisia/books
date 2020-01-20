
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

### 47. IEnumerable<T> 데이터 소스와 IQueryable<T> 데이터 소스를 구분하라 ###

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