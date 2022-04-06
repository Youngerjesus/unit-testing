# 단위 테스트 안티패턴

여기서는 단위 테스트의 안티패턴의 몇가지 예시를 소개하겠다.

## 비공개 메소드 단위 테스트 

비공개 메소드만을 위한 단위 테스트를 작성하는 건 올바르지 않다. 

주로 구현의 세부사항과 관련 있어서 좋은 테스트의 요소인 리팩터링 내성이라는 지표가 부족하기 때문에. 

단위 테스트는 식별할 수 있는 동작 단위로 검증을 해야한다.

근데 만약에 식별할 수 있는 동작 단위로 단위 테스트를 짰는데 비공개 메소드에서는 충분한 커버리지를 보여주지 못하고 있고 비공개 메소드가 복잡하며 중요한 로직을 포함하고 있다면 어떻게 해야할까? 

이 상황 자체가 문제인데 충분한 커버리지가 나오지 않고 있다면 필요없는 코드가 있을 수 있다. 

또 다른 문제로 비공개 메소드가 복잡하고 중요한 로직을 포함하고 있다면 그건 추상화가 충분히 되지 않은 코드일 수 있다. 

그러므로 다른 클래스로 뺴서 협력하도록 해야한다.

비공개 메소드의 테스트가 타당한 경우는 비공개 메소드 자체가 식별할 수 있는 동작일 때이다. 

이게 이해가 되지 않을 수 있는데 ORM 같은 도구를 사용하면 데이터베이스에서 조회한 후 객체를 생성해주니까 도메인 모델은 비공개 생성자로 해두는 경우가 있다. 

이러면 이 객체는 직접 생성할 수 없으므로 테스트하기 어려워진다. 이런 경우는 비공개 생성자를 공개 생성자로 바꿔도 충분히 문제가 되지 않기 때문에 돌려도 된다.

## 비공개 상태 노출

비공개 상태가 변경되었다는 걸 테스트하고 싶다.

이때는 어떻게 해야할까? 테스트를 위해서 이 비공개 상태를 공개 상태로 바꾸는 건 캡슐화를 위반하는 행동이다. 

이때는 비공개 상태를 이용하는 식별할 수 있는 동작을 기반으로 테스트를 검사하면 된다. 

즉 다음과 같은 에시를 기준으로는 `promote()` 메소드 호출 후 상태가 변경될 것인데 이때는 `GetDiscount()` 메소드를 통해서 변경된 상태에 따른 동작을 검증해보면 된다.

```csharp
public class Customer
{
    private CustomerStatus _status = CustomerStatus.Regular;
    
    public void Promote()
    {
        _status = CustomerStatus.Preferred;
    }
    
    public decimal GetDiscount()
    {
        return _status == CustomerStatus.Preferred ? 0.05m : 0m;
    }
}

public enum CustomerStatus
{
    Regular,
    Preferred
}
```

## 테스트로 유출된 도메인 지식 

도메인 지식과 세부 구현 사항을 테스트로 노출하는 것도 또 하나의 안티 패턴이다.

다음과 같이 계산 알고리즘이 있다고 예로 들어보자. 

```csharp
public static class Calculator
{
    public static int Add(int value1, int value2)
    {
        return value1 + value2;
    }
}
```

```csharp
public class CalculatorTests
{
    [Fact]
    public void Adding_two_numbers()
    {
        int value1 = 1;
        int value2 = 3;
        int expected = value1 + value2; // 구현 노출

        int actual = Calculator.Add(value1, value2);
        Assert.Equal(expected, actual);
    }
}
```

이런 경우에는 검증할 값을 그냥 하드 코딩으로 명시하는 것이 좋다. 객체인 경우에는 직접 만들던가. 

```csharp
public class CalculatorTests
{
    [Theory]
    [InlineData(1, 3, 4)]
    [InlineData(11, 33, 44)]
    [InlineData(100, 500, 600)]
    public void Adding_two_numbers(int value1, int value2, int expected) {
        int actual = Calculator.Add(value1, value2);
        Assert.Equal(expected, actual);
    }
}
```

## 코드 오염

제품 코드안에 테스트를 위한, 테스트 환경을 위한 코드가 있다면 이것도 안티 패턴이다. 

이런 코드는 혼란을 줄 수 있고 유지비용을 높인다.

## 구체 클래스를 목으로 처리하기 

지금까지 목을 사용하는 방법은 인터페이스를 목으로 처리하는 방법이다.

하지만 클래스의 경우에도 목으로 처리할 수 있다. 

이를 통해 불필요한 기능은 구현하고 필요한 기능은 그대로 사용하는 방식을 적용할 수 있다. 

하지만 이렇게 목을 통해 구체 클래스를 처리한다는 자체가 SRP 를 위반하고 있다는 뜻이기도 하다. 

해당 객체의 기능이 많으니까 이런식으로 목으로 처리하는 것이다.

이 경우에는 클래스를 쪼개면 된다. 예시로보자. 

```csharp
public class StatisticsCalculator
{
    public (double totalWeight, double totalCost) Calculate(
        int customerId)
    {
        List<DeliveryRecord> records = GetDeliveries(customerId);
        double totalWeight = records.Sum(x => x.Weight);
        double totalCost = records.Sum(x => x.Cost);
        return (totalWeight, totalCost);
    }
    
    public List<DeliveryRecord> GetDeliveries(int customerId)
    {
        /* 프로세스 외부 의존성을 용한 기능으로 부작용을 야기할 수 있는 메소드 */
    }
}
```

- `StatisticCalculator` 는 특정 고객에게 배달된 모든 배송들의 무게와 비용 같은 고객 정보 통계를 수집하고 계산한다. 

- 이 클래스를 목으로 대체할 때 프로세스 외부 의존성을 사용하는 `GetDeliveries()` 기능을 대체한다. 

통합 테스트를 작성할 때 해당 객체를 그대로 넣는다면 부작용을 일으킬 수 있다. 그래서 목으로 대체해야 한다고 생각할 수 있다. 

하지만 이렇게 하는 경우는 SRP 를 위반하는 거니까 이 문제는 프로세스 외부 의존성을 사용하는 부분을 객체로 쪼개서 처리하면 된다.

코드로 보자. 

```csharp
public class DeliveryGateway : IDeliveryGateway
{
    public List<DeliveryRecord> GetDeliveries(int customerId)
    {
        /* Call an out-of-process dependency
        to get the list of deliveries */
    }
}

public class StatisticsCalculator
{
    public (double totalWeight, double totalCost) Calculate(
        List<DeliveryRecord> records)
    {
        double totalWeight = records.Sum(x => x.Weight);
        double totalCost = records.Sum(x => x.Cost);
        return (totalWeight, totalCost);
    }
}
```

이렇게 하면 IDeliveryGateway 를 목으로 대체해서 사용하면 깔끔하게 구현하는게 가능하다. 

## 시간 처리하기 

시간 같은 특정 시점에 따라서 값이 달라지는 경우는 테스트하기 어렵다. 

특히 정적 필드나 정적 메소드 (= LocalDateTime.now()) 와 같은 메소드를 이용할 때 특히 어렵다. 

이 경우에는 시간을 만들어주는 서비스를 만들고 의존성으로 넣어줘서 시간을 생성하도록 하거나

메소드를 실행할 때 시간을 인자로 받아서 처리하도록 하면 테스트하기 쉽다.





