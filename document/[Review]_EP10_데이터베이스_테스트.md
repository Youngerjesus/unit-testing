# 데이터베이스 테스트 

통합 테스트에서는 데이터베이스를 목으로 사용하지 않고 실제로 접근해서 사용한다. 

이렇게 사용할 때 준비하기 위한 전제들을 알아보고 데이터베이스에서 트랜잭션을 관리하는 방법과 테스트를 위한 데이터를 관리하는 방법 그리고 통합 테스트에서 유지 보수 비용을 줄이는 것, 마지막으로 일반적으로 데이터베이스 테스트를 할 때 물어보는 질문에 답하면서 마무리하겠다. 

## 데이터베이스 테스트 전제 조건

- Git 과 같은 형상 관리 시스템에 데이터베이스 스키마를 기록하는 것. (히스토리를 알 수 있도록)

- 개발자를 위해 데이터베이스 인스턴스를 개발자 머신에서 사용하는 것. (데이터베이스를 공유해서 사용하면 문제가 생길 여지가 많음. 주로 데이터베이스에서의 테스트는 데이터베이스에 들어간 데이터를 기반으로 테스트하기 때문에.)

- 데이터베이스 배포에 마이그레이션 기반 방식을 사용하는 것. (마이그레이션 방식을 사용하면 데이터 모션 문제를 방지할 수 있다. 데이터 모션은 새로운 데이터 스키마를 따르도록 하는 문제다.)

## 데이터베이스 트랜잭션 관리 

어플리케이션에서 데이터베이스와 관련된 작업을 할 땐 데이터베이스 작업 단위인 트랜잭션을 고려해야한다. 

여기서 고려해야 한다는 건 원자적 (Atomic) 한 연산을 어디까지 보장할 것인가? 를 묻는 것이다. 

예로 고객이 회사 이메일로 변경하면 이메일 변경 작업과 회사 이메일을 사용하는 고객의 수를 증가하는 작업은 같이해야 한다. 

이를 별도의 트랜잭션으로 관리하면 데이터 모순이 일어날 수 있다.

트랜잭션을 관리하면 불필요한 데이터베이스 호출이 줄어들어서 좋다. 

번외로 관계형 데이터베이스의 경우에는 트랜잭션을 통해서 데이터 모순을 피하기 쉬운 편이지만 비관계형 데이터베이스의 경우에는 트랜잭션이 없고 단건의 문서에서만 아토믹한 연산을 주기도 한다. 

그래서 여러 건의 문서를 집계해서 사용하는 일은 지양해야한다. 

통합 테스트에서도 트랜잭션을 사용할 때 주의해야하는데 다음과 같이 트랜잭션을 재사용하면 안된다. 

```csharp 
[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    using (var context = new CrmContext(ConnectionString))
    {
        // Arrange
        var userRepository =
            new UserRepository(context);
        var companyRepository =
            new CompanyRepository(context);
        var user = new User(0, "user@mycorp.com",
            UserType.Employee, false);
        
        userRepository.SaveUser(user);
        
        var company = new Company("mycorp.com", 1);
        companyRepository.SaveCompany(company);
        context.SaveChanges();
        
        var busSpy = new BusSpy();
        var messageBus = new MessageBus(busSpy);
        var loggerMock = new Mock<IDomainLogger>();
        var sut = new UserController(
            context,
            messageBus,
            loggerMock.Object);

        // Act
        string result = sut.ChangeEmail(user.UserId, "new@gmail.com");
        
        // Assert
        Assert.Equal("OK", result);
        
        User userFromDb = userRepository
            .GetUserById(user.UserId);
            
        Assert.Equal("new@gmail.com", userFromDb.Email);
        Assert.Equal(UserType.Customer, userFromDb.Type);

        Company companyFromDb = companyRepository
            .GetCompany();

        Assert.Equal(0, companyFromDb.NumberOfEmployees);

        busSpy.ShouldSendNumberOfMessages(1)
            .WithEmailChangedMessage(user.UserId, "new@gmail.com");

        loggerMock.Verify(
            x => x.UserTypeHasChanged(
                user.UserId, UserType.Employee, UserType.Customer),
            Times.Once);
    }
}
```

- 일반적으로 제품 코드는 트랜잭션을 사용하고 폐기한다. 하지만 여기서는 여러 검증 구문에서 트랜잭션을 재사용한다. 

- ORM 같은 도구를 사용할 경우 성능을 위해서 캐시로 데이터를 보관하는 경우가 있는데 이러면 데이터베이스에 보관된 데이터 상태를 테스트하지 않는 문제가 생긴다. 

- 그러므로 별도의 트랜잭션을 적용하는게 좋다. 


## 테스트 데이터 생명주기  

통합 테스트에서 데이터베이스를 공용으로 사용할 경우 각 통합 테스트는 서로 분리하기 어렵다. 

각 통합 테스트마다 데이터를 지우고 넣고를 반복하면서 영향을 줄 수 있기 떄문에.

그러므로 통합 테스트는 조금 느리더라도 순차적으로 실행해야 한다. 

또 통합 테스트는 데이터베이스 데이터의 상태에 의존하면 안된다. 

그러므로 데이터를 늘 데이터베이스에서 지워줘야 하는데 이는 테스트를 시작할 때 지우는게 가장 좋다. (마지막에 지운다면 중간에 작업이 실패하면 더미 데이터가 데이터베이스에 남게된다.)  

다른 얘기로 성능의 문제 때문에 운영 환경의 데이터베이스 대신에 인메모리 데이터베이스를 사용하는 경우가 많은데 추천하지 않는다. (똑같은 환경을 맞추는게 유의미하기 때문에.)

## 테스트 구절에서 코드 재사용하기 

통합 테스트는 코드가 보편적으로 단위 테스트보다 많은데 (협력자 수가 많고 검증할 게 많을 수 있다.) 이 경우 코드의 가독성이 떨어지고 유지 보수 비용을 증가시킬 수 있다. 

그래서 중복되는 코드 같은 경우는 재사용하는 걸 권장한다. 

### 준비 구절에서 코드 재사용하기 

테스트를 위한 데이터를 생성하는 과정을 private 팩토리 메소드로 대체하는걸 추천한다. 

그리고 이런 팩토리 메소드가 여러개 생긴다면 별도의 헬퍼 클래스를 만들자.

예시로는 다음과 같다. 

```csharp
User user = CreateUser(
            email: "user@mycorp.com",
            type: UserType.Employee);
```

이렇게 테스트 픽스처 (테스트 실행 대상) 을 생성하는 방식의 패턴을 오브젝트 마더 (Object Mother) 라고 부른다.

이렇게 생성하는 것과 빌더 패턴을 이용해서 생성하는 방식이 있는데 빌더 패턴은 상용구가 너무 많이 필요하다. 


### 실행 구절에서 코드 재사용하기 

모든 통합 테스트 구절에서 실행 구절이 중복되는 부분이 있다면 이를 재사용할 수 있다. 

이거는 데코레이터 패턴을 이용하는 것으로 예제를 통해서 보자.

```csharp
string result;
using (var context = new CrmContext(ConnectionString))
{
    var sut = new UserController(
        context, messageBus, loggerMock.Object);
    
    // Act
    result = sut.ChangeEmail(user.UserId, "new@gmail.com");
}
```

이 코드를 재사용 하도록 줄일 수 있다.

```csharp
// Delegate defines a controller function.
private string Execute(
    Func<UserController, string> func,
    MessageBus messageBus,
    IDomainLogger logger)
{
    using (var context = new CrmContext(ConnectionString))
    {
        var controller = new UserController(context, messageBus, logger);
        return func(controller);
    }
}
```

````csharp
string result = Execute(
    x => x.ChangeEmail(user.UserId, "new@gmail.com"),
    messageBus, loggerMock.Object);
````

### 검증 구절에서 코드 재사용하기 

검증 구절에서도 준비 구절과 같이 검증할 데이터를 생성하는 부분을 헬퍼 메소드를 통해서 줄일 수 있다.

더 나아가서 Spy 를 통해서 검증을 줄이는 방법과 같이 기존 도메인 클래스 위에 플루언트 인터페이스를 기반으로 검증할 수도 있다.

```csharp
public static class UserExternsions
{
    public static User ShouldExist(this User user)
    {
        Assert.NotNull(user);
        return user;
    }
    public static User WithEmail(this User user, string email)
    {
        Assert.Equal(email, user.Email);
        return user;
    }
}
```

```csharp
User userFromDb = QueryUser(user.UserId);
userFromDb
    .ShouldExist()
    .WithEmail("new@gmail.com")
    .WithType(UserType.Customer);
```

## 데이터베이스 테스트에 대한 일반적인 질문들

### 읽기 테스트를 해야하는가? 

일반적으로 읽기 작업보다는 쓰기 작업이 중요하다. 잘못된 데이터가 들어갔을 때 생기는 버그가 더 크기 때문에.

그리고 읽기 작업은 도메인 클래스 레이어와 협력할 일이 없다. 

일반적으로 읽기 작업은 성능을 고려한 쿼리로 짜는 경우가 많기 떄문에.

그래서 읽기 작업만을 위한 테스트 보다는 통합 테스트로 가는게 훨씬 낫다. 아니면 가장 중요한 읽기 테스트만 테스트하자. 

### 리포지터리 테스트를 해야하는가? 

이것도 결론만 말하면 통합 테스트로 포함시키는게 훨씬 낫다. 

리포지토리 테스트는 간단한 테스트라서 가치가 별로 없기도 하고 