# Unit testing tips by examples in PHP

## Table of Contents

1. [Introduction](#introduction)
2. [Test doubles](#test-doubles)
3. [Naming](#naming)
4. [AAA pattern](#aaa-pattern)
5. [Object mother](#object-mother)
6. [Parameterized test](#parameterized-test)
7. [Two schools of unit testing](#two-schools-of-unit-testing)
   * [Classical](#classical)
   * [Mockist](#mockist)
   * [Dependencies](#dependencies)
8. [Mock vs Stub](#mock-vs-stub)
9. [Three styles of unit testing](#three-styles-of-unit-testing)
    * [Output](#output)
    * [State](#state)
    * [Communication](#communication)
10. [Functional architecture and tests](#functional-architecture-and-tests)
11. [Observable behaviour vs implementation details](#observable-behaviour-vs-implementation-details)
12. [Unit of behaviour](#unit-of-behaviour)
13. [Humble pattern](#humble-pattern)
14. [Trivial test](#trivial-test)
15. [Fragile test](#fragile-test)
16. [Test fixtures](#test-fixtures)
17. [General testing anti-patterns](#general-testing-anti-patterns)
    * [Exposing private state](#exposing-private-state)
    * [Leaking domain details](#leaking-domain-details)
    * [Mocking concrete classes](#mocking-concrete-classes)
    * [Testing private methods](#testing-private-methods)
    * [Time as a volatile dependency](#time-as-a-volatile-dependency)

## Introduction

## Test doubles

Test doubles are fake dependencies used in tests.

### Stubs

A dummy is a just simple implementation which does nothing.

#### Dummy

```php
final class Mailer implements MailerInterface
{
    public function send(Message $message): void
    {
    }
}
```

A fake is a simplified implementation to simulate the original behaviour.

#### Fake

```php
final class InMemoryCustomerRepository implements CustomerRepositoryInterface
{
    /**
     * @var Customer[]
     */
    private array $customers;

    public function __construct()
    {
        $this->customers = [];
    }

    public function store(Customer $customer): void
    {
        $this->customers[(string) $customer->id()->id()] = $customer;
    }

    public function get(CustomerId $id): Customer
    {
        if (!isset($this->customers[(string) $id->id()])) {
            throw new CustomerNotFoundException();
        }

        return $this->customers[(string) $id->id()];
    }

    public function findByEmail(Email $email): Customer
    {
        foreach ($this->customers as $customer) {
            if ($customer->getEmail()->isEqual($email)) {
                return $customer;
            }
        }

        throw new CustomerNotFoundException();
    }
}
```

A stub is the simplest implementation with a hardcoded behaviour.

#### Stub

```php
final class UniqueEmailSpecificationStub implements UniqueEmailSpecificationInterface
{
    public function isUnique(Email $email): bool
    {
        return true;
    }
}
```

```php
$specificationStub = $this->createMock(UniqueEmailSpecificationInterface::class);
$specificationStub->method('isUnique')->willReturn(true);
```

### Mocks

A spy is an implementation to verify a specific behaviour.

#### Spy
```php
final class Mailer implements MailerInterface
{
    /**
     * @var Message[]
     */
    private array $messages;
    
    public function __construct()
    {
        $this->messages = [];
    }

    public function send(Message $message): void
    {
        $this->messages[] = $message;
    }

    public function getCountOfSentMessages(): int
    {
        return count($this->messages);
    }
}
```

#### Mock

A mock is a configured imitation to verify calls on a collaborator.

```php
$message = new Message('test@test.com', 'Test', 'Test test test');
$mailer = $this->createMock(MailerInterface::class);
$mailer
    ->expects($this->once())
    ->method('send')
    ->with($this->equalTo($message));
```

:exclamation: 
To verify incoming interactions use a stub, but to verify outcoming interactions use a mock. 
More: [Mock vs Stub](#mock-vs-stub)

## Naming

:heavy_minus_sign: Not good:
```php
public function test(): void
{
    $subscription = SubscriptionMother::new();

    $subscription->activate();

    self::assertEquals(Status::activated(), $subscription->status());
}
```

:heavy_check_mark: **Specify explicitly what are you testing**
```php
public function sut(): void
{
    // sut = System under test
    $sut = SubscriptionMother::new();

    $sut->activate();

    self::assertEquals(Status::activated(), $sut->status());
}
``` 

:heavy_minus_sign: Not good:
```php
public function it_throws_invalid_credentials_exception_when_sign_in_with_invalid_credentials(): void
{

}

public function testCreatingWithATooShortPasswordIsNotPossible(): void
{

}

public function testDeactivateASubscription(): void
{

}
```

:heavy_check_mark: Better:  
- **Using underscore improves readability**
- **The name should describe the behaviour, not the implementation**
- **Use names without technical keywords. It should be readable for non-programmer person.**

```php
public function sign_in_with_invalid_credentials_is_not_possible(): void
{

}

public function creating_with_a_too_short_password_is_not_possible(): void
{

}

public function deactivating_an_activated_subscription_is_valid(): void
{

}

public function deactivating_an_inactive_subscription_is_invalid(): void
{

}
```

:information_source: Describing the behaviour is important in testing the domain scenarios. 
If your code is just a utility one it's less important.

## AAA pattern

It's also common Given, When, Then.

:heavy_check_mark: Separate three sections of the test:  

- **Arrange**: Bring the system under test in the desired state. Prepare dependencies, arguments and finally construct
the SUT.
- **Act**: Invoke a tested element.
- **Assert**: Verify the result, the final state or the communication with collaborators.

```php
public function aaa_pattern_example_test(): void
{
    //Arrange|Given
    $sut = SubscriptionMother::new();

    //Act|When
    $sut->activate();

    //Assert|Then
    self::assertEquals(Status::activated(), $sut->status());
}
```

## Object mother

The pattern helps to create specific objects which can be reused in a few tests. Because of that the arrange section
is concise and the test as a whole is more readable.

```php
final class SubscriptionMother
{
    public static function new(): Subscription
    {
        return new Subscription();
    }

    public static function activated(): Subscription
    {
        $subscription = new Subscription();
        $subscription->activate();
        return $subscription;
    }

    public static function deactivated(): Subscription
    {
        $subscription = self::activated();
        $subscription->deactivate();
        return $subscription;
    }
}
```

```php
final class ExampleTest
{
    public function example_test_with_activated_subscription(): void
    {
        $activatedSubscription = SubscriptionMother::activated();

        // do something

        // check something
    }

    public function example_test_with_deactivated_subscription(): void
    {
        $deactivatedSubscription = SubscriptionMother::deactivated();

        // do something

        // check something
    }
}
```

## Parameterized test

The parameterized test is a good option to test the SUT with a lot of parameters without repeating the code.  

:thumbsdown: This kind of test is less readable. To increase a little the readability negative and positive examples should be
split up to different tests.

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     * @dataProvider getInvalidEmails
     */
    public function detects_an_invalid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertEquals(false, $result);
    }

    /**
     * @test
     * @dataProvider getValidEmails
     */
    public function detects_an_valid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertEquals(true, $result);
    }

    public function getInvalidEmails(): array
    {
        return [
            ['test'],
            ['test@'],
            ['test@test'],
            //...
        ];
    }

    public function getValidEmails(): array
    {
        return [
            ['test@test.com'],
            ['test123@test.com'],
            ['Test123@test.com'],
            //...
        ];
    }
}
```

## Two schools of unit testing

### Classical

```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_subscription_with_can_always_suspend_policy_is_always_possible(): void
    {
        $canAlwaysSuspendPolicy = new CanAlwaysSuspendPolicy();
        $sut = new Subscription();

        $result = $sut->suspend($canAlwaysSuspendPolicy);

        self::assertTrue($result);
        self::assertEquals(Status::suspend(), $sut->status());
    }
}
```

### Mockist

```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_subscription_with_can_always_suspend_policy_is_always_possible(): void
    {
        $canAlwaysSuspendPolicy = $this->createMock(SuspendingPolicyInterface::class);
        $canAlwaysSuspendPolicy->method('suspend')->willReturn(true);
        $sut = new Subscription();

        $result = $sut->suspend($canAlwaysSuspendPolicy);

        self::assertTrue($result);
        self::assertEquals(Status::suspend(), $sut->status());
    }
}
```

### Dependencies

[TODO]

## Mock vs Stub

Example: 
```php
final class NotificationService
{
    public function __construct(
        private MailerInterface $mailer,
        private MessageRepositoryInterface $messageRepository
    ) {}

    public function send(): void
    {
        $messages = $this->messageRepository->getAll();
        foreach ($messages as $message) {
            $this->mailer->send($message);
        }
    }
}
```

:x: Bad:

- **Asserting interactions with stubs leads to fragile tests**

```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function sends_all_notifications(): void
    {
        $message1 = new Message();
        $message2 = new Message();
        $messageRepository = $this->createMock(MessageRepositoryInterface::class);
        $messageRepository->method('getAll')->willReturn([$message1, $message2]);
        $mailer = $this->createMock(MailerInterface::class);
        $sut = new NotificationService($mailer, $messageRepository);

        $messageRepository->expects(self::once())->method('getAll');
        $mailer->expects(self::exactly(2))->method('send')
            ->withConsecutive([self::equalTo($message1)], [self::equalTo($message2)]);

        $sut->send();
    }
}
```

:heavy_check_mark: Good:
```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function sends_all_notifications(): void
    {
        $message1 = new Message();
        $message2 = new Message();
        $messageRepository = $this->createMock(MessageRepositoryInterface::class);
        $messageRepository->method('getAll')->willReturn([$message1, $message2]);
        $mailer = $this->createMock(MailerInterface::class);
        $sut = new NotificationService($mailer, $messageRepository);

        // Removed an asserting interactions with the stub
        $mailer->expects(self::exactly(2))->method('send')
            ->withConsecutive([self::equalTo($message1)], [self::equalTo($message2)]);

        $sut->send();
    }
}
```

## Three styles of unit testing

### Output

:heavy_check_mark:  
- **The best resistance to refactoring**
- **The best accuracy**
- **The lowest cost of maintainability**  
- **If it is possible you should prefer this kind of test**

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     * @dataProvider getInvalidEmails
     */
    public function detects_an_invalid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertEquals(false, $result);
    }

    /**
     * @test
     * @dataProvider getValidEmails
     */
    public function detects_an_valid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertEquals(true, $result);
    }

    public function getInvalidEmails(): array
    {
        return [
            ['test'],
            ['test@'],
            ['test@test'],
            //...
        ];
    }

    public function getValidEmails(): array
    {
        return [
            ['test@test.com'],
            ['test123@test.com'],
            ['Test123@test.com'],
            //...
        ];
    }
}
```

### State

:white_check_mark:
- **Worse resistance to refactoring**
- **Worse accuracy**
- **Higher cost of maintainability**

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     */
    public function adding_an_item_to_cart(): void
    {
        $item = new CartItem('Product');
        $sut = new Cart();

        $sut->addItem($item);

        self::assertEquals(1, $sut->getCount());
        self::assertEquals($item, $sut->getItems()[0]);
    }
}
```

### Communication

:white_check_mark:
- **The worst resistance to refactoring**
- **The worst accuracy**
- **The highest cost of maintainability**

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     */
    public function sends_all_notifications(): void
    {
        $message1 = new Message();
        $message2 = new Message();
        $messageRepository = $this->createMock(MessageRepositoryInterface::class);
        $messageRepository->method('getAll')->willReturn([$message1, $message2]);
        $mailer = $this->createMock(MailerInterface::class);
        $sut = new NotificationService($mailer, $messageRepository);

        $mailer->expects(self::exactly(2))->method('send')
            ->withConsecutive([self::equalTo($message1)], [self::equalTo($message2)]);

        $sut->send();
    }
}
```

## Functional architecture and tests

## Observable behaviour vs implementation details

## Unit of behaviour

## Humble pattern

## Trivial test

## Fragile test

## Test fixtures

## Testing

## General testing anti-patterns

### Exposing private state

### Leaking domain details

### Mocking concrete classes

### Testing private methods

### Time as a volatile dependency