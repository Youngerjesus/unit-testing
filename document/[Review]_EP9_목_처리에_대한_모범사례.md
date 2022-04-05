# 목 처리에 대한 모범사례

여기서는 목을 사용할 때의 방법과 목의 가치를 극대화 하는 방법에 대해서 알아보겠다.

목을 사용할 수 있는 경우는 프로세스 외부 의존성 (비관리 의존성) 을 대체할 때, 즉 통합 테스트를 작성하는 경우에만 사용할 수 있다.

이 방침만 가지고 목을 쓰기에는 목의 가치를 최대한으로 이끌어 낼 수 없다.

목을 사용할 때는 시스템의 상호 작용 끝에서 사용하도록 하자.

이 말을 보다 정확하게 이해하기 위해서 예시를 준비했다. (이전 예시와 동일)

```csharp
public class UserController
    {
        private readonly Database _database;
        private readonly EventDispatcher _eventDispatcher;
        public UserController(
            Database database,
            IMessageBus messageBus,
            IDomainLogger domainLogger)
        {
            _database = database;
            _eventDispatcher = new EventDispatcher(messageBus, domainLogger);
    }
    
    public string ChangeEmail(int userId, string newEmail)
    {
        object[] userData = _database.GetUserById(userId);
        User user = UserFactory.Create(userData);
        
        string error = user.CanChangeEmail();
        if (error != null)
           return error;
        
        object[] companyData = _database.GetCompany();
        Company company = CompanyFactory.Create(companyData);
        
        user.ChangeEmail(newEmail, company);
        
        _database.SaveCompany(company);
        _database.SaveUser(user);
        _eventDispatcher.Dispatch(user.DomainEvents);
        return "OK";
    }
}
```

유저의 이메일 변경사항을 프로세스 외부 의존성인 메시지 버스에 보내도록 한다.

이 메시지 버스는 목으로 대체할 수 있다.

그래서 우리는 통합 테스트를 이렇게 작성할 것이다.

```csharp
[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    // Arrange
    var db = new Database(ConnectionString);
    User user = CreateUser("user@mycorp.com", UserType.Employee, db);
    
    CreateCompany("mycorp.com", 1, db);
    var messageBusMock = new Mock<IMessageBus>();
    var loggerMock = new Mock<IDomainLogger>();
    var sut = new UserController(
            db, messageBusMock.Object, loggerMock.Object);

    // Act
    string result = sut.ChangeEmail(user.UserId, "new@gmail.com");

    // Assert
    Assert.Equal("OK", result);
    object[] userData = db.GetUserById(user.UserId);
    User userFromDb = UserFactory.Create(userData);
    Assert.Equal("new@gmail.com", userFromDb.Email);
    Assert.Equal(UserType.Customer, userFromDb.Type);
    
    object[] companyData = db.GetCompany();
    Company companyFromDb = CompanyFactory.Create(companyData);
    Assert.Equal(0, companyFromDb.NumberOfEmployees);

    messageBusMock.Verify(
        x => x.SendEmailChangedMessage(
            user.UserId, "new@gmail.com"),
        Times.Once);

    loggerMock.Verify(
        x => x.UserTypeHasChanged(
            user.UserId,
            UserType.Employee,
            UserType.Customer),
        Times.Once);
}
```

- 메시지를 날리는 MessageBus 를 목으로 대체하고 상호작용을 검증한다.

MessageBus 의 구조를 한번 보자.

```csharp
public interface IMessageBus
{
    void SendEmailChangedMessage(int userId, string newEmail);
}

public class MessageBus : IMessageBus
{
    private readonly IBus _bus;
    
    public void SendEmailChangedMessage(int userId, string newEmail)
    {
        _bus.Send("Type: USER EMAIL CHANGED; " +
            $"Id: {userId}; " +
            $"NewEmail: {newEmail}");
    } 
}

public interface IBus
{
    void Send(string message);
}
```

- MessageBus 는 IBus 라는 메시지를 보내는 SDK 의 사용자 정의 Wrapper 클래스를 내부에 가지고 있다.

- IBus 는 메시지를 보낼 때 사용자 연결 자격 증명과 같은 세부적인 구현을 숨기기 위해서 작성한 클래스다.

- IMessageBus 는 IBus 를 좀 더 추상화해서 도메인과 관련된 정보를 남기도록 헀다.

이전에 말한 시스템의 상호 작용 끝에서 목을 사용해야 한다면 우리는 여기서 MessageBus 가 아닌 IBus 에 목을 사용해야한다.

그 이유로는 IBus 에서 목을 사용할 경우 코드 범위를 극한까지 검증하는게 가능하다.

또 다른 이유로는 결국에 메시지를 보내는 부분은 IBus 를 통해서 보낸다.

즉 MessageBus 보다 IBus 가 더 변경될 이유가 적다.

정리하자면 리팩터링 내성과 회귀방지 측면을 따져볼 때 시스템의 상호작용 끝에서 목을 쓰는게 좋다.

그렇다면 시스템의 끝은 어디까지일까?

프로세스 외부 의존성을 사용하는 경우 해당 의존성을 직접적으로 사용하는 경우보다 래핑 (Wrapping) 해서 사용하는 걸 권장한다.

그렇게하면 구현을 숨기는 캡슐화 측면과 라이브러리의 업데이트나 변경에 따른 전파를 막을 수 있다는 측면, 마지막으로 도메인 언어를 이용할 수 있다는 장점이 있다.

시스템의 끝은 이 래핑 클래스까지를 말한다.

이 래핑 클래스를 어댑터 (Adapter) 식으로 사용하도록 하고 이 어댑터를 목으로 대체하는 걸 권장한다.

## 목을 스파이로 대체하는 기법

목을 수동으로 작성한 걸 스파이라고 한다.

스파이는 목을 검증할 때 일반 목보다 나을 수 있는데 예시로 보자.

위의 `IBus` 를 스파이로 구현한 예시다.

```csharp
public class BusSpy : IBus
{
    private List<string> _sentMessages = new List<string>();
        
    public void Send(string message)
    {
        _sentMessages.Add(message);
    }

    public BusSpy ShouldSendNumberOfMessages(int number)
    {
        Assert.Equal(number, _sentMessages.Count);
        return this;
    }

    public BusSpy WithEmailChangedMessage(int userId, string newEmail)
    {
        string message = "Type: USER EMAIL CHANGED; " +
            $"Id: {userId}; " +
            $"NewEmail: {newEmail}";
        
        Assert.Contains(
            _sentMessages, x => x == message);
        return this;
    }
}
```

```csharp
[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    var busSpy = new BusSpy();
    var messageBus = new MessageBus(busSpy);
    var loggerMock = new Mock<IDomainLogger>();
    var sut = new UserController(db, messageBus, loggerMock.Object);
    /* ... */
    
    busSpy.ShouldSendNumberOfMessages(1)
        .WithEmailChangedMessage(user.UserId, "new@gmail.com");
}
```

- 즉 스파이를 통해서 좀 더 꼼꼼한 검증을 하는게 가능해지고 추상화를 시킬 수 있다.

## 목 검증할 때 주의사항

목을 검증할 땐 `상호작용을 했다.` 정도만 검증하지 말고 더 자세하게 검증하도록 하자.

이 메소드를 몇 번 호출헀다던지, 해당 메소드 외의 따른 메소드는 호출하지 않았다던지 등을 검증하도록 하자. 
