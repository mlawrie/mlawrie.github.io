---
layout: post
title:  "Really Simple Dependency Injection in TypeScript"
date:   2016-05-24 16:36:24 -0600
permalink: simple-typescript-dependency-injection
---

I've been working on a [side project](https://github.com/mlawrie/sticky-board) in TypeScript. The goal is to eventually build a product similar to [IdeaBoardz](http://www.ideaboardz.com/) which is better-suited to distributed retros. I'm also using it as a platform for exploring type safety in JavaScript and some other new concepts.

Almost immediately, I was frustrated with what's available for mocking objects in tests. In 'regular' JavaScript I've gotten used to using a tool like [proxyquireify](https://github.com/thlorenz/proxyquireify), [rewire-webpack](https://github.com/jhnns/rewire-webpack) or the built-in mocking capability of [Jest](https://facebook.github.io/jest/). The problem is that none of these methods provide any sort of type-safety for TypeScript (I suspect Jest may be capable of this-- I didn't investigate) since they all just rewire dependencies.

With that in mind, I think TypeScript is really more suited to Dependency Injection (DI). The problem is that none of the DI libraies I investigated look particularly appealing. Many require that all injected dependencies are classes either explicitly or implicitly by requiring that dependencies are injected by annotations that only work in the context of classes. Almost all of them involve some sort of global dependency container in which all dependencies are registered. A single file which imports the entire application is really a non-starter on larger projects that use some sort of transpilation  since that'll result in very long build times.

## The solution: Really Simple Dependency Injection in TypeScript

Given that this is a side project with no deadlines, I embarked on rethinking DI for TypeScript. After a few attempts, I came up with the below code. It's 14 lines and suffers from none of the issues I mentioned: it can inject any instance of anything, it's type-safe and it doesn't need a global dependency container. It's also a little different than most DI libraries which I'll explain below.

{% highlight javascript %}
// injector.ts
let mocks:{[id: string] : () => any} = {}

export const injectMock = <T>(factory: () => T, mock: () => T) => {
  mocks[factory.toString()] = mock
}
  
export const clearMocks = () => {
  mocks = {}
}

export const mockable = <T>(factory: () => T): T => {
  if (typeof mocks[factory.toString()] !== 'undefined') {
    return mocks[factory.toString()]()
  }
  return factory()
}
{% endhighlight %}

Here's an example of it's use:

{% highlight javascript %}
// someFunction.ts
import { mockable } from 'injector'
import { someDependency } from 'someDependency'
export const someFunction = () =>
 mockable(() => someDependency).getNumber() * 2


// someFunction.test.ts
import { mock } from 'injector'
import { someDependency } from 'someDependency'
import { someFunction } from 'someFunction'

describe('someFunction' => {
  it('returns the number multiplied by 2', () => {
    let mockDependency = { getNumber: () => 5 }
    
    mock<someDependency>(() => someDependency, () => mockDependency)
    // mock is called like this:
    // mock<Dependency>([dependency factory method], [mock factory method])
    
    expect(someFunction()).to.eql(10)
  })
})

{% endhighlight %}

It's really simple, right? You can also inject and mock anything you wish:

{% highlight javascript %}
const myClass = mockable(() => new MyClass())
// myClass instanceof MyClass === true

const myFunc = mockable(() => someFunc)
// myFunc === someFunc

const someObj = {foo: 'bar'}
const myPlainObject = mockable(() => someObj)
// myPlainObject === someObj
{% endhighlight %}

Critically, the injected dependency's type is defined both in your app and your app's tests! 

## How does it work?

Instead of using a global container to store dependencies, the consumer of the dependency declares it every time it's used. The injector then compares this declaration to any mocks which have been registered. If a mock matches, the mock is returned instead of the real dependency.

When the application code runs, the injector does nothing and always returns the dependency.

It relies on calling `toString()` on functions so it is not compatible with minification in your tests. I can't imagine why anyone would minify their tests, though. You can definitely minify your production code, though.

There is one caveat: You must declare your depenencies exactly the same way in tests as in your app code. This means you can't call something that you inject `foo` in tests and `Foo` in your app's code. Personally, I think this is a small price for the simplicity and scalability of this solution. 