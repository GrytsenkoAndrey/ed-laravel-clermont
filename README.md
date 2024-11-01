# Content

- [Mork partial test](#test-mock-partial)
- [Test old()](#test-old-helper)
- [Avoid unnecessary work when querying relationships](#unnecessary-work-when-querying-relationships)
- [Generating slugs in factories](#generating-slugs-in-factories)
- [Swagger | API documentation](#swagger-api-doc)
- [echo print difference](#echo-print-difference)
- [solid and patterns](#solid-patterns)


## Test mock partial

A partial mock lets you mock specific methods on a class, but allows the class to otherwise work as it normally would. This can be a useful testing technique.

And Laravel, helpful as always, provides a helper on the base TestCase to create partial mocks:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-test-mock-partial/assets/63291871/1b6fee8c-21d0-407e-9740-f0d150118496)

One thing the docs don't mention is that a partial mock created this way will not run the constructor of the class being mocked.

This can be confusing at first, because you might assume the constructor would run as normal. Our intention is only to mock one or two specific methods, after all.

The underlying Mockery library made the decision that you don't want the constructor to be called, even for a partial mock, because it might have some unintended side effects. So this is the default behavior for partial mocks, and this is what the Laravel test helper does.

But if you really need the constructor to run, you can accomplish this by using a different array-based syntax:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-test-mock-partial/assets/63291871/7a52fb63-c533-4410-a7ce-f2c2ee0a0f1b)

Mockery calls this a "partial test double" and refers to the other default kind of partial mock as a "runtime partial double".

There is no Laravel test helper to generate this. If you need your constructor to run on a partial mock, you'll have to manually call Mockery with this syntax instead.

## Test old helper

I was writing a test for a Blade component recently, and there was some value handling in the constructor I wanted to cover. This component was meant for user input, and you could specify a default value, but it would also fall back to old() form input if present.

There are a couple ways you could test this. One approach would be to set up the world where you have data in the session which the old() helper will look for. Then you could call your component, render it, and make assertions against the output.

But this involves a fair amount of setup since that session comes from the active request. And if I want to do a simple Blade component test, I don't even need a request.

Another approach would be to simply assert that the old() helper was called with the expected field name and default value fallback. The advantage here is there's nothing to set up and no request to generate.

I don't think there's one universally right answer here. Some of it boils down to a matter of taste and what else you might be doing in your test or component.

In my case, I chose the second approach. Which leads to the question: How do you do it?

The old() helper is on the Request, so you might start by mocking that Request class. I won't go into all the details here, but this is a bad idea, and will lead you down a difficult path.

Here's how I did it instead:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/9426684e-5ec1-4de8-b63a-044ec14faa6e)

The spy doesn't try to mock out the Request class, it just monitors calls into its methods, enabling us to make assertions about them later.

With this test, I'm confident that my Blade component constructor was properly considering old form input, but I don't have to set up a bunch of session data to prove it.

## Unnecessary work when querying relationships

I was working in a controller that was queried by a bunch of receipt printers looking for pending jobs.

The goal was to improve reliability and performance. I saw some code that was looking to see if there was a pending job:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/18bdba8b-d3c7-4f37-b736-35f11936154e)

The $pendingJob was only being used as a boolean. The actual model was never used for anything.

At first, I was going to do this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/3c634b4a-e6c8-4c86-948a-4ab7d7af2a8c)

This has a bug though, since the printJobs property is a collection, and there is no exists() method on a collection.

So then my next thought was to do this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/15cc6e22-552b-49ed-8255-5c48843c9b19)

It is true that isNotEmpty works on a collection, and has the same basic meaning as exists, but this is not good code.

Why? If the whole point was to avoid loading and then not using print jobs, I have made absolutely no progress on that goal. I've basically changed nothing about the query.

The SQL query run in both cases looks something like this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/35607dae-b5d0-468b-9dda-0afc8ce53e51)

Notice how we're still querying for all the print jobs, and then Eloquent will hydrate each one of these rows into a model. All of that takes time and memory.

A much better solution is to add a filter to the relationship method:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/a3acfb8d-8f2b-42bf-b2e6-2f0c4b0d4e9f)

Notice how I'm now using the relationship method instead of the property, so it runs a much more efficient query.

Now the SQL query looks something like this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/31ef3cfb-18f5-4ac4-9234-a97e90d02391)

So instead of returning all the rows from the database and unnecessarily hydrating models, my query returns a single row with a single boolean value.

This might seem like a small thing, but in my case the code was in an endpoint getting polled every couple seconds by a fleet of printers, so this small improvement actually had a significant impact.

## Generating slugs in factories

Normally, I would generate a slug for an Eloquent model with an observer or an event. The creating event can verify if the slug field is filled, and if not, use the Str helper to add it to the current creation payload. Pretty cool.

But, sometimes you can't do it like this. Perhaps it's a legacy project, events are turned off in your tests, or any number of reasons. So, when using Eloquent factories, you might find yourself doing something like this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/14db35b4-b323-4d86-93fc-b6a43527f153)

I don't know about you, but I really don't want to do that. Seems repetitive - especially since I know I want the slug to be generated directly from the title field.

Laravel provides a solution, though. When you pass a closure to a field definition in the factory declaration, you get access to the other attributes that are being used to create the model. So, let's update our factory definition to this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/fc1c9f03-ad21-4a64-b863-1c0166fa23ea)

Now, our slug is generated automatically from our title. (If we want a different one, we can still pass it in using the create() method.)

So, now, we can do this:

![image](https://github.com/GrytsenkoAndrey/ed-laravel-clermont/assets/63291871/b842bcd5-a022-41ae-9b50-e46c71f4c4af)


## Swagger API doc

Thinking back over the many Laravel projects I've joined over the years, I can only think of a small number of times when the project had a formal API specification to document API endpoints it provided. Most times, the API was only for internal use, whether that be for code in the front-end to call, or to support a separate mobile app.

Today, I'd like to make the case that there is value in having an API specification even if your API is not used by any external applications.

The first benefit I have found is around internal communication both between developers and also between a developer and the person specifying business requirements. When a developer reads new feature requirements, it is tempting to dive right into the implementation, but taking the time to first write up an API specification for any endpoints you need can really help you ensure you're building the right thing.

For example, just thinking through the types of responses an endpoint can return can often guide you through some important decisions.

- Can it return a 401 Unauthorized or a 403 Forbidden?
- What are the rules around who can access this endpoint?
- What authorization logic should be enforced?
- Can it return a 422 Unprocessable Content error?
- What is the validation logic for each piece of data a user can send in to that endpoint?
- This field is a datetime, but what is the desired format?
- This field is a string, but what is the maximum length? You're going to save it in a database, and the field probably has a max length in the schema, but there might be other business rules to enforce as well.

I also like how writing an API specification forces some larger architectural questions, and gets you thinking early about how to handle versioning. Perhaps your application is multi-tenant. How should the current tenant be specified? Is it a header? A path value? Is it consistent between all your endpoints?

For your endpoints that return collections, how are they paginated? How do you filter them? Can you specify how much detail you want for related records? All of these things are formalized in an API specification, and just taking the time to explicitly write it out can help your API be much more consistent in its design.

Hopefully, I've convinced you it's worth further consideration. Tomorrow, I'll get more tactical and talk about how to actually get started building the specification.

As you go, update your feature tests to add API spec validation. For Laravel apps, I like the [openapi-httpfoundation-testing package](https://click.convertkit-mail.com/8kuo65mppdtoh0mx5lrikhz778g99u3/n2hohqu3qnpo3pi0/aHR0cHM6Ly9naXRodWIuY29tL29zdGVlbC9vcGVuYXBpLWh0dHBmb3VuZGF0aW9uLXRlc3Rpbmc=). This will keep you honest that your spec is accurately describing the requests and responses in your application.


## echo print difference

Decades ago, it was common to use **echo** in PHP applications. In fact, we used it so much, we had a special short PHP tag <​?= to make it easier to type.

Today, though, it's pretty rare to see echo in a modern Laravel application.

One place it does get used is when streaming a download as your response.

For example, streaming a CSV file might look like this:

![image](https://github.com/user-attachments/assets/a4a0f7ba-0648-4483-bb26-74285877319d)

With that setup out of the way, now to the question of the article: when might you need to use **print** instead of **echo**?

What if you want to refactor that anonymous function to an arrow function? In this example, **echo** will throw a parse error.

**Parse error: syntax error, unexpected token "echo"**

Why? Because the arrow function body must be a single expression that returns a value. echo has a void return type and never returns a value.

On the other hand, **print** always produces the return value **1**, so you can use it.

This code works (and looks nicer, if you ask me):

![image](https://github.com/user-attachments/assets/8aab4853-7dc7-4bd4-b41c-326890bce5b4)


## SOLID patterns

Принципы SOLID помогают в проектировании гибких и масштабируемых систем. Каждый из них часто связан с определёнными паттернами проектирования, которые помогают реализовать эти принципы. Рассмотрим их:

- S — Single Responsibility Principle (Принцип единственной ответственности):
  
        Описание: Каждый класс должен иметь только одну причину для изменения, т.е. он должен отвечать за одну конкретную задачу.
  
        Паттерны:
          - Factory: Избавляет объекты от ответственности за создание экземпляров других классов.
          - Strategy: Позволяет выделить алгоритмы в отдельные классы, чтобы каждая реализация имела свою ответственность.

- O — Open/Closed Principle (Принцип открытости/закрытости):
  
        Описание: Классы должны быть открыты для расширения, но закрыты для модификации.
  
        Паттерны:
          - Decorator: Позволяет добавлять новую функциональность объектам без изменения их исходного кода.
          - Template Method: Открывает возможность наследникам переопределять отдельные части алгоритма, не изменяя сам алгоритм.

- L — Liskov Substitution Principle (Принцип подстановки Барбары Лисков):
  
        Описание: Объекты должны быть заменяемы экземплярами их подклассов без изменения корректности программы.
  
        Паттерны:
          - Adapter: Позволяет классам с несовместимыми интерфейсами работать вместе, соблюдая при этом принцип подстановки.
          - Bridge: Отделяет абстракцию от её реализации, что позволяет подклассам изменять или расширять функциональность.

- I — Interface Segregation Principle (Принцип разделения интерфейса):
  
        Описание: Клиенты не должны зависеть от интерфейсов, которые они не используют.
  
        Паттерны:
          - Facade: Создает простой интерфейс к сложной системе, позволяя скрывать ненужную функциональность.
          - Proxy: Предоставляет интерфейс к другому объекту, скрывая его сложную структуру или поведение.

- D — Dependency Inversion Principle (Принцип инверсии зависимостей):
  
        Описание: Модули верхнего уровня не должны зависеть от модулей нижнего уровня; оба должны зависеть от абстракций.
  
        Паттерны:
          - Dependency Injection: Позволяет передавать зависимости в объект извне, тем самым уменьшая жёсткую связь между компонентами.
          - Abstract Factory: Создаёт интерфейс для создания семейств связанных объектов, избавляя модули от зависимости от конкретных классов.














