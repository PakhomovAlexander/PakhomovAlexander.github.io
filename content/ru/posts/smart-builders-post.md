+++ 
date = 2020-07-08T16:40:00+03:00
title = "На пути к TDD. SmartBuilder паттерн"
description = "На пути к TDD. SmartBuilder паттерн"
slug = "smart-builder" 
tags = ["java", "tdd", "dsl", "JUnit5"]
categories = ["На пути к TDD"]
externalLink = ""
series = []
+++

# На пути к TDD. SmartBuilder паттерн.

## Зачем

Действительно, зачем? Зачем делать серию статей о [TDD](https://ru.wikipedia.org/wiki/Разработка_через_тестирование)? 

- :loudspeaker: На конференциях об этом говорят. 
- :mortar_board: На собеседовании пояснить за TDD могут. 
- :muscle: Кто-то даже пробовал. 
- :heart: Кому-то даже понравилось. 

А кто пробовал использовать TDD на большом, закостенелом проекте?  Где размер объектов подогревает стулья, а после прочтения ТЗ хочется выйти в окно. Ну и как? Работает тест дривен магия? Скорее всего нет. :shit:

Расскажу как не вываливаться из цикла red-green-refactor на том самом большом проекте.

---

## Что мешает 

Для комфортного кодирования в потоке red-green-refactor 
тесты должны быть небольшого размера, методы простыми для восприятия.  

На словах просто, а как насчет кода? 

### Исходные данные

Допустим доменная модель состоит из `Person` и `Organisation`. На деле в разы больше, но для примера достаточно.

```java
public abstract class AbstractCustomer {
    private Address address;

    public abstract String getWelcomeName();

    public String getCountry() {
        return address.getCountry();
    }

    // geters and setters
}
```

```java
public class Organisation extends AbstractCustomer {

    private String organisationName;

    @Override
    public String getWelcomeName() {
        return organisationName;
    }


    // geters and setters
}
```

```java
public class Person extends AbstractCustomer {
    private String firstName;
    private String lastName;

    @Override
    public String getWelcomeName() {
        return firstName + " " + lastName;
    }


    // geters and setters
}
```

```java
public class Address {
    private String street;
    private String city;
    private String country;
    private int house;
    private int index;

    // getters and setters
}
```

Для создания объектов используется паттерн JavaBeans. Это когда конструктор пустой, а объект настраивается через сеттеры. Да, не лучший выбор, но и проект с историческим наследием. [Вот статья](https://medium.com/@muradhajiyev/why-builder-pattern-not-telescoping-or-javabeans-6daa689f418) о разных видах создания объектов в java. Спойлер: используйте билдер.

Мы будем работать с тем, что есть. :information_desk_person:

#### Задача

Разработать сервис Hostel. Hostel умеет приветствовать клиентов фразой `"Welcome in hoselName, customer.getWelcomeName(). customer.getCountry() is great!"`. Вот и вся задача.

Погнали!

### Первый заход

Определяем интерфейс Hostel

```java 
public interface Hostel {
    String getWelcome(AbstractCustomer customer);
}
```

И дефолтную имплементацию 

``` java 
public class HostelImpl implements Hostel {
    private String hostelName;

    public HostelImpl(String hostelName) {
        this.hostelName = hostelName;
    }

    @Override
    public String getWelcome(AbstractCustomer customer) {
        return null;
    }
```

Пишем тест

```java 
class HostelImplTestWithoutSmartBuilder {
    Hostel service;

    @BeforeEach
    void setUp() {
        service = new HostelImpl("Best hostel");
    }

    @Test
    void shouldWelcomePerson() {
        // Given
        Address address = new Address();
        address.setCountry("Russia");
        address.setCity("Moscow");
        // And
        Person person = new Person();
        person.setFirstName("Alexander");
        person.setLastName("Pakhomov");
        person.setAddress(address);

        // When
        String welcomeMessage = service.getWelcome(person);

        // Then
        assertThat(welcomeMessage).isEqualTo("Welcome in Best hostel, Alexander Pakhomov. Russia is great!");
    }
```

Предлагаю остановиться и посмотреть на тест. 
Для такого простого кейса слишком много кода. Проблема с секцией `//Given`.

```java{linenos=table,hl_lines=["11-19"],linenostart=1}
class HostelImplTestWithoutSmartBuilder {
    Hostel service;

    @BeforeEach
    void setUp() {
        service = new HostelImpl("Best hostel");
    }

    @Test
    void shouldWelcomePerson() {
        // Given
        Address address = new Address();
        address.setCountry("Russia");
        address.setCity("Moscow");
        // And
        Person person = new Person();
        person.setFirstName("Alexander");
        person.setLastName("Pakhomov");
        person.setAddress(address);

        // When
        String welcomeMessage = service.getWelcome(person);

        // Then
        assertThat(welcomeMessage).isEqualTo("Welcome in Best hostel, Alexander Pakhomov. Russia is great!");
    }
```
Слишком громоздко. Суть теста замыливается созданием объектов `Person` и `Address`. Нужно это упростить.

### Второй заход

#### Выносим создание объектов в отдельный метод

Мысль очевидная и в большинстве случаев работaет. Но когда дело доходит до огромного количества полей у объектов, то получаем что-то вроде 

```java 
Person givenPerson = createSamplePerson("Saha", "Pushkin", "Russia","Kolotushkina 29", null, null, 0, "")
```

и это еще лайт. Напомню, проект с историческим наследием! 

Да, код можно улучшить: вынести параметры в переменные, добавить еще методов. Но это не решение проблемы, это подавление симптомов. Нужно использовать другой механизм создания объектов.

### Третий заход

#### Используем Builder

```java
// Given
var person = Person.builder()
                    .firstName("Alexander")
                    .lastName("Pakhomov")
                    .address(Address.builder()
                                    .country("Russia")
                                    .city("Moscow")
                                    .build())
                    .build()
        );
```

Уже лучше. Параметры именованны и типизированны. Но что тут не так? 

Мы хотим писать в TDD парадигме. Тесты должны быть простыми, отражать только спецификацию. Мы должны читать тесты как описание поведения объекта. Наличие `Person.builder()` и никому (в тесте) не нужных вызовов `.build()` усложняет чтение кода. 

```java{linenos=table,hl_lines=[2,5,8,9],linenostart=1}
// Given
var person = Person.builder()
                    .firstName("Alexander")
                    .lastName("Pakhomov")
                    .address(Address.builder()
                                    .country("Russia")
                                    .city("Moscow")
                                    .build())
                    .build()
        );
```

---

## Решение -- SmartBuilder :construction_worker:

### Как это выглядит в итоге 

```java 
// Given
var person = given(
        person().firstName("Alexander")
                .lastName("Pakhomov")
                .address(address().country("Russia").city("Moscow"))
        );
```

### Имплементация 

Базовый интерфейс для всех SmartBuilder-ов

```java 
public interface GenericBuilder<T, B extends GenericBuilder<T, B>> {
    T build();
}
```

Абстрактный билдер для `AbstractiCustormer`

```java
public abstract class CustomerBuilder<T extends AbstractCustomer, B extends CustomerBuilder<T, B>>
        implements GenericBuilder<T,B> {

    private Address address;

    public GenericBuilder<T,B> address(Address address) {
        this.address = address;
        return this;
    }

    public GenericBuilder<T,B> address(AddressBuilder<Address> addressBuilder) {
        this.address = addressBuilder.build();
        return this;
    }

    protected abstract T createCustomer();

    @Override
    public T build() {
        T customer = createCustomer();
        customer.setAddress(address);

        return customer;
    }
}
```

Билдер для `Person`
```java
public class PersonBuilder<T extends Person> extends CustomerBuilder<T, PersonBuilder<T>> {
    private String firstName;
    private String lastName;

    public PersonBuilder<T> firstName(String firstName) {
        this.firstName = firstName;
        return this;
    }

    public PersonBuilder<T> lastName(String lastName) {
        this.lastName = lastName;
        return this;
    }

    @Override
    protected T createCustomer() {
        T person = (T) new Person();
        person.setFirstName(this.firstName);
        person.setLastName(this.lastName);

        return person;
    }
}
```

Билдер для `Organisation`

```java 
public class OrganisationBuilder<T extends Organisation> extends CustomerBuilder<T, OrganisationBuilder<T>> {
    private String organisationName;

    public OrganisationBuilder<T> organisationName(String organisationName) {
        this.organisationName = organisationName;
        return this;
    }

    @Override
    protected T createCustomer() {
        T organisation = (T) new Organisation();
        organisation.setOrganisationName(organisationName);

        return organisation;
    }
}
```

И финалочка - DSL в виде статичных методов

```java 
public class GenericBuilderDsl {
    public static <T, B extends GenericBuilder<T, B>> T given(GenericBuilder<T, B> builder) {
        return builder.build();
    }

    public static PersonBuilder<Person> person() {
        return new PersonBuilder<>();
    }

    public static OrganisationBuilder<Organisation> organisation() {
        return new OrganisationBuilder<>();
    }

    public static AddressBuilder<Address> address() {
        return new AddressBuilder<>();
    }
}
```

Вот и все! SmartBuilder-ы удобны для написания тестов, которые может прочитать даже ваш PM. 

Если вам понравилась эта идея: 
- [вот имплементация](https://github.com/PakhomovAlexander/java-tests-best-practices/tree/master/dsl-for-tests/src/test/java/io/github/pakhomovalexander/builders)
- [тут](https://github.com/PakhomovAlexander/java-tests-best-practices/blob/master/dsl-for-tests/src/test/java/io/github/pakhomovalexander/service/HostelImplTestWithoutSmartBuilder.java) можно посмотреть тест без SmartBuilder
- [тут](https://github.com/PakhomovAlexander/java-tests-best-practices/blob/master/dsl-for-tests/src/test/java/io/github/pakhomovalexander/service/HostelImplTestWithSmartBuilder.java) тест с SmartBuilder

Подписывайтесь на [душный канал](https://t.me/toxic_enterprise) и [твиттер](https://twitter.com/alexandr_phmv). В следующей статье пойдем дальше. :metal: