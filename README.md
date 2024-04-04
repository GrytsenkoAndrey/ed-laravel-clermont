# ed-laravel-test-mock-partial

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



