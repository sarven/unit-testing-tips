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

### Classical (Detroit school)

- The unit as a single unit of behaviour, it can be a few related classes. 
- Every test should be isolated from others. So it must be possible to invoke them in parallel or in a any order.

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

### Mockist (London school)

- The unit as a single class.
- The unit should be isolated from all collaborators.

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

:information_source:
**The classical approach is better in order to avoid fragile tests.**

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

:heavy_check_mark:  The best option:  
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

:white_check_mark: Worse option:  
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

:white_check_mark: The worst option: 
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

:x: Bad:
```php
final class NameService
{
    public function __construct(private CacheStorageInterface $cacheStorage) {}

    public function loadAll(): void
    {
        $namesCsv = array_map('str_getcsv', file(__DIR__.'/../names.csv'));
        $names = [];

        foreach ($namesCsv as $nameData) {
            if (!isset($nameData[0], $nameData[1])) {
                continue;
            }

            $names[] = new Name($nameData[0], new Gender($nameData[1]));
        }

        $this->cacheStorage->store('names', $names);
    }
}
```

**How to test a code like this? It is possible only with an integration test, because there is directly used 
an infrastructure code related with a file system.**

:heavy_check_mark: Good:
```php
final class NameParser
{
    /**
     * @param array $namesData
     * @return Name[]
     */
    public function parse(array $namesData): array
    {
        $names = [];

        foreach ($namesData as $nameData) {
            if (!isset($nameData[0], $nameData[1])) {
                continue;
            }

            $names[] = new Name($nameData[0], new Gender($nameData[1]));
        }

        return $names;
    }
}
```

```php
final class CsvNamesFileLoader
{
    public function load(): array
    {
        return array_map('str_getcsv', file(__DIR__.'/../names.csv'));
    }
}
```

```php
final class ApplicationService
{
    public function __construct(
        private CsvNamesFileLoader $fileLoader,
        private NameParser $parser,
        private CacheStorageInterface $cacheStorage
    ) {}

    public function loadNames(): void
    {
        $namesData = $this->fileLoader->load();
        $names = $this->parser->parse($namesData);
        $this->cacheStorage->store('names', $names);
    }
}
```

```php
final class ValidUnitExampleTest extends TestCase
{
    /**
     * @test
     */
    public function parse_all_names(): void
    {
        $namesData = [
            ['John', 'M'],
            ['Lennon', 'U'],
            ['Sarah', 'W']
        ];
        $sut = new NameParser();

        $result = $sut->parse($namesData);
        self::assertEquals(
            [
                new Name('John', new Gender('M')),
                new Name('Lennon', new Gender('U')),
                new Name('Sarah', new Gender('W'))
            ],
            $result
        );
    }
}
```

## Observable behaviour vs implementation details

:x: Bad:

```php
final class ApplicationService
{
    public function __construct(private SubscriptionRepositoryInterface $subscriptionRepository) {}

    public function renewSubscription(int $subscriptionId): bool
    {
        $subscription = $this->subscriptionRepository->findById($subscriptionId);

        if (!$subscription->getStatus()->isEqual(Status::expired())) {
            return false;
        }

        $subscription->setStatus(Status::active());
        $subscription->setModifiedAt(new \DateTimeImmutable());
        return true;
    }
}
```

```php
final class Subscription
{
    private Status $status;

    private \DateTimeImmutable $modifiedAt;

    public function __construct(Status $status, \DateTimeImmutable $modifiedAt)
    {
        $this->status = $status;
        $this->modifiedAt = $modifiedAt;
    }

    public function getStatus(): Status
    {
        return $this->status;
    }

    public function setStatus(Status $status): void
    {
        $this->status = $status;
    }

    public function getModifiedAt(): \DateTimeImmutable
    {
        return $this->modifiedAt;
    }

    public function setModifiedAt(\DateTimeImmutable $modifiedAt): void
    {
        $this->modifiedAt = $modifiedAt;
    }
}
```

```php
final class InvalidTestExample extends TestCase
{
    /**
     * @test
     */
    public function renew_an_expired_subscription_is_possible(): void
    {
        $modifiedAt = new \DateTimeImmutable();
        $expiredSubscription = new Subscription(Status::expired(), $modifiedAt);
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($expiredSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        self::assertEquals(Status::active(), $expiredSubscription->getStatus());
        self::assertGreaterThan($modifiedAt, $expiredSubscription->getModifiedAt());
        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function renew_an_active_subscription_is_not_possible(): void
    {
        $modifiedAt = new \DateTimeImmutable();
        $activeSubscription = new Subscription(Status::active(), $modifiedAt);
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($activeSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        self::assertEquals($modifiedAt, $activeSubscription->getModifiedAt());
        self::assertFalse($result);
    }
}
```

:heavy_check_mark: Good:

```php
final class ApplicationService
{
    public function __construct(private SubscriptionRepositoryInterface $subscriptionRepository) {}

    public function renewSubscription(int $subscriptionId): bool
    {
        $subscription = $this->subscriptionRepository->findById($subscriptionId);
        return $subscription->renew(new \DateTimeImmutable());
    }
}
```

```php
final class Subscription
{
    private Status $status;

    private \DateTimeImmutable $modifiedAt;

    public function __construct(\DateTimeImmutable $modifiedAt)
    {
        $this->status = Status::new();
        $this->modifiedAt = $modifiedAt;
    }

    public function renew(\DateTimeImmutable $modifiedAt): bool
    {
        if (!$this->status->isEqual(Status::expired())) {
            return false;
        }

        $this->status = Status::active();
        $this->modifiedAt = $modifiedAt;
        return true;
    }

    public function active(\DateTimeImmutable $modifiedAt): void
    {
        //simplified
        $this->status = Status::active();
        $this->modifiedAt = $modifiedAt;
    }

    public function expire(\DateTimeImmutable $modifiedAt): void
    {
        //simplified
        $this->status = Status::expired();
        $this->modifiedAt = $modifiedAt;
    }

    public function isActive(): bool
    {
        return $this->status->isEqual(Status::active());
    }
}
```

```php
final class ValidTestExample extends TestCase
{
    /**
     * @test
     */
    public function renew_an_expired_subscription_is_possible(): void
    {
        $expiredSubscription = SubscriptionMother::expired();
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($expiredSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        // skip checking modifiedAt as it's not a part of observable behaviour. To check this value we
        // would have to add getter for modifiedAt, probably only for tests purposes.
        self::assertTrue($expiredSubscription->isActive());
        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function renew_an_active_subscription_is_not_possible(): void
    {
        $activeSubscription = SubscriptionMother::active();
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($activeSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        self::assertTrue($activeSubscription->isActive());
        self::assertFalse($result);
    }
}
```

## Unit of behaviour

:x: Bad:

```php
class CannotSuspendExpiredSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isExpired()) {
            return false;
        }

        return true;
    }
}
```

```php
class CannotSuspendExpiredSubscriptionPolicyTest extends TestCase
{
    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_expired(): void
    {
        $policy = new CannotSuspendExpiredSubscriptionPolicy();
        $subscription = $this->createMock(Subscription::class);
        $subscription->method('isExpired')->willReturn(true);

        self::assertFalse($policy->suspend($subscription, new \DateTimeImmutable()));
    }

    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_not_expired(): void
    {
        $policy = new CannotSuspendExpiredSubscriptionPolicy();
        $subscription = $this->createMock(Subscription::class);
        $subscription->method('isExpired')->willReturn(false);

        self::assertTrue($policy->suspend($subscription, new \DateTimeImmutable()));
    }
}
```

```php
class CannotSuspendNewSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isNew()) {
            return false;
        }

        return true;
    }
}
```

```php
class CannotSuspendNewSubscriptionPolicyTest extends TestCase
{
    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_new(): void
    {
        $policy = new CannotSuspendNewSubscriptionPolicy();
        $subscription = $this->createMock(Subscription::class);
        $subscription->method('isNew')->willReturn(true);

        self::assertFalse($policy->suspend($subscription, new \DateTimeImmutable()));
    }

    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_not_new(): void
    {
        $policy = new CannotSuspendNewSubscriptionPolicy();
        $subscription = $this->createMock(Subscription::class);
        $subscription->method('isNew')->willReturn(false);

        self::assertTrue($policy->suspend($subscription, new \DateTimeImmutable()));
    }
}
```

```php
class CanSuspendAfterOneMonthPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        $oneMonthEarlierDate = \DateTime::createFromImmutable($at)->sub(new \DateInterval('P1M'));

        return $subscription->isOlderThan(\DateTimeImmutable::createFromMutable($oneMonthEarlierDate));
    }
}
```

```php
class CanSuspendAfterOneMonthPolicyTest extends TestCase
{
    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable('2021-01-29');
        $policy = new CanSuspendAfterOneMonthPolicy();
        $subscription = new Subscription(new \DateTimeImmutable('2020-12-28'));

        self::assertTrue($policy->suspend($subscription, $date));
    }

    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_not_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable('2021-01-29');
        $policy = new CanSuspendAfterOneMonthPolicy();
        $subscription = new Subscription(new \DateTimeImmutable('2020-01-01'));

        self::assertTrue($policy->suspend($subscription, $date));
    }
}
```

```php
class Status
{
    private const EXPIRED = 'expired';
    private const ACTIVE = 'active';
    private const NEW = 'new';
    private const SUSPENDED = 'suspended';

    private string $status;

    private function __construct(string $status)
    {
        $this->status = $status;
    }

    public static function expired(): self
    {
        return new self(self::EXPIRED);
    }

    public static function active(): self
    {
        return new self(self::ACTIVE);
    }

    public static function new(): self
    {
        return new self(self::NEW);
    }

    public static function suspended(): self
    {
        return new self(self::SUSPENDED);
    }

    public function isEqual(self $status): bool
    {
        return $this->status === $status->status;
    }
}
```

```php
class StatusTest extends TestCase
{
    public function testEquals(): void
    {
        $status1 = Status::active();
        $status2 = Status::active();

        self::assertTrue($status1->isEqual($status2));
    }

    public function testNotEquals(): void
    {
        $status1 = Status::active();
        $status2 = Status::expired();

        self::assertFalse($status1->isEqual($status2));
    }
}
```

```php
class SubscriptionTest extends TestCase
{
    /**
     * @test
     */
    public function suspending_a_subscription_is_possible_when_a_policy_returns_true(): void
    {
        $policy = $this->createMock(SuspendingPolicyInterface::class);
        $policy->expects($this->once())->method('suspend')->willReturn(true);
        $sut = new Subscription(new \DateTimeImmutable());

        $result = $sut->suspend($policy, new \DateTimeImmutable());

        self::assertTrue($result);
        self::assertTrue($sut->isSuspended());
    }

    /**
     * @test
     */
    public function suspending_a_subscription_is_not_possible_when_a_policy_returns_false(): void
    {
        $policy = $this->createMock(SuspendingPolicyInterface::class);
        $policy->expects($this->once())->method('suspend')->willReturn(false);
        $sut = new Subscription(new \DateTimeImmutable());

        $result = $sut->suspend($policy, new \DateTimeImmutable());

        self::assertFalse($result);
        self::assertFalse($sut->isSuspended());
    }

    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable();
        $futureDate = $date->add(new \DateInterval('P1M'));
        $sut = new Subscription($date);

        self::assertTrue($sut->isOlderThan($futureDate));
    }

    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_not_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable();
        $futureDate = $date->add(new \DateInterval('P1D'));
        $sut = new Subscription($date);

        self::assertTrue($sut->isOlderThan($futureDate));
    }
}
```

:heavy_check_mark: Good:

```php
final class CannotSuspendExpiredSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isExpired()) {
            return false;
        }

        return true;
    }
}
```

```php
final class CannotSuspendNewSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isNew()) {
            return false;
        }

        return true;
    }
}
```

```php
final class CanSuspendAfterOneMonthPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        $oneMonthEarlierDate = \DateTime::createFromImmutable($at)->sub(new \DateInterval('P1M'));

        return $subscription->isOlderThan(\DateTimeImmutable::createFromMutable($oneMonthEarlierDate));
    }
}
```

```php
final class Status
{
    private const EXPIRED = 'expired';
    private const ACTIVE = 'active';
    private const NEW = 'new';
    private const SUSPENDED = 'suspended';

    private string $status;

    private function __construct(string $status)
    {
        $this->status = $status;
    }

    public static function expired(): self
    {
        return new self(self::EXPIRED);
    }

    public static function active(): self
    {
        return new self(self::ACTIVE);
    }

    public static function new(): self
    {
        return new self(self::NEW);
    }

    public static function suspended(): self
    {
        return new self(self::SUSPENDED);
    }

    public function isEqual(self $status): bool
    {
        return $this->status === $status->status;
    }
}
```

```php
final class Subscription
{
    private Status $status;

    private \DateTimeImmutable $createdAt;

    public function __construct(\DateTimeImmutable $createdAt)
    {
        $this->status = Status::new();
        $this->createdAt = $createdAt;
    }

    public function suspend(SuspendingPolicyInterface $suspendingPolicy, \DateTimeImmutable $at): bool
    {
        $result = $suspendingPolicy->suspend($this, $at);
        if ($result) {
            $this->status = Status::suspended();
        }

        return $result;
    }

    public function isOlderThan(\DateTimeImmutable $date): bool
    {
        return $this->createdAt < $date;
    }

    public function activate(): void
    {
        $this->status = Status::active();
    }

    public function expire(): void
    {
        $this->status = Status::expired();
    }

    public function isExpired(): bool
    {
        return $this->status->isEqual(Status::expired());
    }

    public function isActive(): bool
    {
        return $this->status->isEqual(Status::active());
    }

    public function isNew(): bool
    {
        return $this->status->isEqual(Status::new());
    }

    public function isSuspended(): bool
    {
        return $this->status->isEqual(Status::suspended());
    }
}
```

```php
final class SubscriptionSuspendingTest extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_expired_subscription_with_cannot_suspend_expired_policy_is_not_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());
        $sut->activate();
        $sut->expire();

        $result = $sut->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertFalse($result);
    }

    /**
     * @test
     */
    public function suspending_a_new_subscription_with_cannot_suspend_new_policy_is_not_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertFalse($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_new_policy_is_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());
        $sut->activate();

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_expired_policy_is_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());
        $sut->activate();

        $result = $sut->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_subscription_before_a_one_month_is_not_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable('2020-01-01'));

        $result = $sut->suspend(new CanSuspendAfterOneMonthPolicy(), new \DateTimeImmutable('2020-01-10'));

        self::assertFalse($result);
    }

    /**
     * @test
     */
    public function suspending_an_subscription_after_a_one_month_is_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable('2020-01-01'));

        $result = $sut->suspend(new CanSuspendAfterOneMonthPolicy(), new \DateTimeImmutable('2020-02-02'));

        self::assertTrue($result);
    }
}
```

## Humble pattern

[TODO]

## Trivial test

:x: Bad:

```php
final class Customer
{
    public function __construct(private string $name) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }
}
```

```php
final class CustomerTest extends TestCase
{
    public function testSetName(): void
    {
        $customer = new Customer('Jack');

        $customer->setName('John');

        self::assertEquals('John', $customer->getName());
    }
}
```

```php
final class EventSubscriber
{
    public static function getSubscribedEvents(): array
    {
        return ['event' => 'onEvent'];
    }

    public function onEvent(): void
    {

    }
}
```

```php
final class EventSubscriberTest extends TestCase
{
    public function testGetSubscribedEvents(): void
    {
        $result = EventSubscriber::getSubscribedEvents();

        self::assertEquals(['event' => 'onEvent'], $result);
    }
}
```

## Fragile test

:x: Bad:

```php
final class UserRepository
{
    public function __construct(
        private Connection $connection
    ) {}

    public function getUserNameByEmail(string $email): ?array
    {
        return $this
            ->connection
            ->createQueryBuilder()
            ->from('user', 'u')
            ->where('u.email = :email')
            ->setParameter('email', $email)
            ->execute()
            ->fetch();
    }
}
```

```php
final class TestUserRepository extends TestCase
{
    public function testGetUserNameByEmail(): void
    {
        $email = 'test@test.com';
        $connection = $this->createMock(Connection::class);
        $queryBuilder = $this->createMock(QueryBuilder::class);
        $result = $this->createMock(ResultStatement::class);
        $userRepository = new UserRepository($connection);
        $connection
            ->expects($this->once())
            ->method('createQueryBuilder')
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('from')
            ->with('user', 'u')
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('where')
            ->with('u.email = :email')
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('setParameter')
            ->with('email', $email)
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('execute')
            ->willReturn($result);
        $result
            ->expects($this->once())
            ->method('fetch')
            ->willReturn(['email' => $email]);

        $result = $userRepository->getUserNameByEmail($email);

        self::assertEquals(['email' => $email], $result);
    }
}
```

## Test fixtures

:x: Bad:

```php
final class InvalidTest extends TestCase
{
    private ?Subscription $subscription;

    public function setUp(): void
    {
        $this->subscription = new Subscription(new \DateTimeImmutable());
        $this->subscription->activate();
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_new_policy_is_possible(): void
    {
        $result = $this->subscription->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_expired_policy_is_possible(): void
    {
        $result = $this->subscription->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_a_new_subscription_with_cannot_suspend_new_policy_is_not_possible(): void
    {
        // Here we need to create a new subscription, it is not possible to change $this->subscription to a new subscription
        self::assertTrue(true);
    }
}
```

:heavy_check_mark: Good:

```php
final class ValidTest extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_new_policy_is_possible(): void
    {
        $sut = $this->createAnActiveSubscription();

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_expired_policy_is_possible(): void
    {
        $sut = $this->createAnActiveSubscription();

        $result = $sut->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_a_new_subscription_with_cannot_suspend_new_policy_is_not_possible(): void
    {
        $sut = $this->createANewSubscription();

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertFalse($result);
    }

    private function createANewSubscription(): Subscription
    {
        return new Subscription(new \DateTimeImmutable());
    }

    private function createAnActiveSubscription(): Subscription
    {
        $subscription = new Subscription(new \DateTimeImmutable());
        $subscription->activate();
        return $subscription;
    }
}
```

- It's better to avoid a shared state between tests.
- To reuse elements between a few tests:
    * private factory methods - reusing in one class (like above)
    * [Object mother]((#object-mother)) - reusing in a few classes

## General testing anti-patterns

### Exposing private state

### Leaking domain details

### Mocking concrete classes

### Testing private methods

### Time as a volatile dependency