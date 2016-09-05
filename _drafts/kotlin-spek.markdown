---
layout: post
title:  "Spek - specification test framework for Kotlin"
date:   2016-08-02
categories: Kotlin
tags: java maven spek kotlin
---
I always like the Spock - a specification framework for Java and Groovy. My adventure with Groovy had started from Spock because it was
easier to convince the team to start using new JVM language within project for test code than the production code :) So recently when I had 
started interesting of the new JVM language from JetBrains named Kotlin my idea was the same as some time ago with learning Groovy. Start
learning the Kotlin on the test code - and here comes the Spek!

Before I focus on Spek is worth to say a little more about Kotlin itself. According with <a href="https://en.wikipedia.org/wiki/Kotlin_(programming_language)" target="_blank">Wikipedia</a>:

> Kotlin is a statically-typed programming language that runs on the Java Virtual Machine and also can be compiled to JavaScript source code. 
> Its primary development is from a team of JetBrains programmers based in Saint Petersburg, Russia (the name comes from the Kotlin Island, near 
> St. Petersburg). Kotlin was named Language of the Month in the January 2012 issue of Dr. Dobb's Journal. While not syntax compatible with Java, 
> Kotlin is designed to interoperate with Java code and is reliant on Java code from the existing Java Class Library, such as the collections framework.

From my perspective the first look at Kotlin was very promising. The concise syntax, null safe approach 
<a href="https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare" target="_blank">see why does it matter</a>, 
good interoperability with Java and the fact that behind it is the company which created Intellij Idea force me to deeper look at it. 
So to give it a try on my day-to-day routine I had started looking something similar like Spock. 
And here it is: [Spek](https://jetbrains.github.io/spek/) - a specification framework for JVM written on Kotlin.

Though Spek is written in Kotlin is 100% compatible with Java. The specifications could verify new or existing Java or Kotlin code. What is a specification?
In simple words it is a test in a more human readable way. If someone used the Spock, Jasmine or Mocha then won't have a problem to understand
the Spek. Let's take a look at simple example: 

```java
class CalculatorSpec: Spek({
    given("simple calculator") {
        val calculator = Calculator()

        on("calculating the sum of 2 and 2") {
            val result = calculator.add(2, 2)
            it("should return 4") {
                assertEquals(4, result)
            }
        }
    }
})
```

As you can see on this simple example we have class _CalculatorSpec_ (the Spec postfix is from Specification) which extends the _Spek_ class. Within the body
we could define our specification. When we run the example by Intellij test runner we will see the following result:

![specification example](/assets/images/1/calculator_spec.png)

That kind of testing is much simpler especially if we are talking about readability of the code. We have some subject which based on some action should either
return specific result or execute specific action. That's why the specifications are used as a ubiquitous language on [BDD](https://en.wikipedia.org/wiki/Behavior-driven_development#Specification_as_a_ubiquitous_language).

Ok, so let's take a look a little deeper on the technical side of the Spek. As I mentioned before the _CalculatorSpec_ extends from the open class _Spek_. As the constructor parameter
the _Spek_ class takes the function literal with receiver. What it is? To explain it we have to say something [about the higher order functions and lambdas on Kotlin](https://kotlinlang.org/docs/reference/lambdas.html).

What is a higher order function. According with Kotlin documentation:

> A higher-order function is a function that takes functions as parameters, or returns a function.

So if we have a function type `body: () -> Unit` it’s supposed to be a function that takes no parameters and returns nothing. Just pure action. For example:

```kotlin
fun execute(body: () -> Unit): Unit {
  println("start")
  body()
  println("stop")
}

execute {
    println("function example")
}

// console output:
start
function example
stop

```

Please take a note that in Kotlin, there is a convention that if the last parameter to a function is a function, that parameter can be specified outside of the parentheses.
What is it in this case function literals with receiver? This is a function with a specified receiver object. Inside the body of the function literal, you can call methods on 
that receiver object without any additional qualifiers. If you have ever written any DSL on Groovy I am sure that you know what it is. If not I encourage you to take a deeper look
at Kotlin documentation and examples of their usage of Type-safe Groovy-style builders. You could also read the blog post of the author of Spek framework Hadi Hariri about [that](http://hadihariri.com/2013/01/21/extension-function-literals-in-kotlin-or-how-to-enforce-restrictions-on-your-dsl/)
(it is a little outdated but still worth to review).

The definition of the spek body is the following: `val spekBody: DescribeBody.() -> Unit`. It is a function literal where the receiver object is a object implementing _DescribeBody_ interface.
Let's take a look at _DescribeBody_ interface itself:

```kotlin
interface DescribeBody  {
    fun describe(description: String, evaluateBody: DescribeBody.() -> Unit)
    fun xdescribe(description: String, evaluateBody: DescribeBody.() -> Unit)
    fun fdescribe(description: String, evaluateBody: DescribeBody.() -> Unit)

    fun it(description: String, assertions: () -> Unit)
    fun xit(description: String, assertions: () -> Unit)
    fun fit(description: String, assertions: () -> Unit)

    fun beforeEach(actions: () -> Unit)
    fun afterEach(actions: () -> Unit)

    fun on(description: String, evaluateBody: DescribeBody.() -> Unit) = describe(description, evaluateBody)
    fun given(description: String, evaluateBody: DescribeBody.() -> Unit) = describe(description, evaluateBody)
    fun context(description: String, evaluateBody: DescribeBody.() -> Unit) = describe(description, evaluateBody)
    fun xon(description: String, evaluateBody: DescribeBody.() -> Unit) = xdescribe(description, evaluateBody)
    fun xgiven(description: String, evaluateBody: DescribeBody.() -> Unit) = xdescribe(description, evaluateBody)
    fun xcontext(description: String, evaluateBody: DescribeBody.() -> Unit) = xdescribe(description, evaluateBody)
    fun fon(description: String, evaluateBody: DescribeBody.() -> Unit) = fdescribe(description, evaluateBody)
    fun fgiven(description: String, evaluateBody: DescribeBody.() -> Unit) = fdescribe(description, evaluateBody)
    fun fcontext(description: String, evaluateBody: DescribeBody.() -> Unit) = fdescribe(description, evaluateBody)
}
```

So the interface has the methods which were used on first example in order to create the specification. As can you see the most of the methods have also the function literal with
receiver object _DescribeBody_ - that allows us to nesting the specifications one inside another. With that structure the _JUnitSpekRunner_ could run the tree test. I will try to describe
the specific group of methods from interface:

* _describe, on, given, context_ -> by these methods we could build our nested specifications
* _beforeEach, afterEach_ -> execute specific actions either before or after each test
* _it_ -> part designed to put code assertions, any specification inside this body won't be taken into account by junit spek runner (you won't see it on Intellij runner)

You could also notice that each method has a counterpart with 'x' prefix and 'f' prefix (for example describe has xdescribe and fdescribe). What these methods do?

Let's add to our _CalculatorSpec_ the `xon` part:

```kotlin
given("simple calculator") {
    val calculator = Calculator()

    on("calculating the sum of 2 and 2") {
        val result = calculator.add(2, 2)
        it("should return 4") {
            assertEquals(4, result)
        }
    }

    xon("calculating the multiplication of 2 and 2") {
        todo { "implement multiplication" }
    }
}
```

The result when we run the specification:

![xon](/assets/images/1/calculator_spec_xon.png)

As can you see the test has been skipped. The 'x' prefix means something like ignore the specification part started with 'x' (take into account that could be used with various parts 
of specification like given, context, etc.). The second case is a 'f' prefix. Let's extend our specification:

```kotlin
given("simple calculator") {
    val calculator = Calculator()

    on("calculating the sum of 2 and 2") {
        val result = calculator.add(2, 2)
        it("should return 4") {
            assertEquals(4, result)
        }
    }

    xon("calculating the multiplication of 2 and 2") {
        todo { "implement multiplication" }
    }

    fon("calculating the sum of 2 and 2 and 2") {
        val result = calculator.add(2, 2, 2)
        it("should return 6") {
            assertEquals(6, result)
        }
    }
}
```

Run it:

![fon](/assets/images/1/calculator_spec_fon.png)

As can you see in this case only to 'fon' part has been run. The 'f' prefix means that we should focus only on this part of specification and runner runs only selected test (or maybe I should say
part of specification). Ok - we have reviewed the basic of the Spec and simple example.

Let's try to run different specification. For this case I have prepared example class _TaxRateCalculator_
which takes some _CountryTaxStrategy_ and for specific _TaxPayer_ counts the tax rate. What is important the "production" code has been written on the pure Java. You will find the entire source
code on my github project.

```kotlin
class TaxRateCalculatorSpec : Spek({

    describe("calculating tax") {
        val calculator = TaxRateCalculator(PolandTaxStrategy())

        context("for poland") {
            calculator.taxStrategy = PolandTaxStrategy()

            context("with linear tax payer and 10 000 PLN gross income") {
                val taxPayer = TaxPayer(TaxType.LINEAR, ofPln(10000.00))
                val calculatedTax = calculator.calculateTax(taxPayer)

                it("should return 1900 as tax rate") {
                    assertThat(calculatedTax.tax).isEqualTo(ofPln(1900.00))
                }
                it("should return 8100 as net income") {
                    assertThat(calculatedTax.netIncome).isEqualTo(ofPln(8100.00))
                }
            }

            context("with progressive tax payer and 10 000 PLN gross income") {
                val taxPayer = TaxPayer(TaxType.PROGRESSIVE, ofPln(10000.00))
                val calculatedTax = calculator.calculateTax(taxPayer)

                it("should return 1243.98 as tax rate") {
                    assertThat(calculatedTax.tax).isEqualTo(ofPln(1243.98))
                }
            }

            xcontext("with progressive tax payer and high gross income") {
                //todo implement
            }
        }

        context("for denmark") {
            calculator.taxStrategy = DenmarkTaxStrategy()

            context("with linear tax payer and 10 000 DKK gross income") {
                val taxPayer = TaxPayer(TaxType.LINEAR, ofDkk(10000.00))
                val calculatedTax = calculator.calculateTax(taxPayer)

                it ("should return 2500 DKK as tax rate") {
                    assertThat(calculatedTax.tax).isEqualTo(ofDkk(2500.00))
                }
            }

            context("for linear and progressive tax payer with the same gross income") {
                val grossIncome = ofDkk(12345.67)
                val payerFirst = TaxPayer(TaxType.LINEAR, grossIncome)
                val secondPayer = TaxPayer(TaxType.PROGRESSIVE, grossIncome)
                val taxForFirstPayer = calculator.calculateTax(payerFirst)
                val taxForSecondPayer = calculator.calculateTax(secondPayer)

                it("should have the same tax rate") {
                    assertThat(taxForFirstPayer.tax).isEqualTo(taxForSecondPayer.tax)
                    given("test") {}
                }
            }
        }
    }
})
```
With this expanded example we could see the power of the specification by example. We could easily read the test and figure out the correct 
business logic of the code. In my opinion it is much more easier that _testPolandTaxRate_ method. See the Intellij test runner view:

![tax_rate_spec](/assets/images/1/tax_rate_spec.png)

 


* todo push the code to github repo and put the link
* todo Lack of the @Unroll feature.
* todo add discuss or something like that for the page

Useful links:
* [Spek GitHub](https://github.com/JetBrains/spek)
* 

