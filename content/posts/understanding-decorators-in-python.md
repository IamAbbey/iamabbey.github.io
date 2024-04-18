---
title: 'Understanding Decorators in Python'
date: 2022-02-19T10:11:30+01:00
# weight: 1
# aliases: ["/first"]
tags: ["python"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: https://source.unsplash.com/zbWFT4eVopE
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/IamAbbey/iamabbey.github.io/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
In simple words, Decorators in Python Programming Language are used to add additional functionalities to functions or classes. Decorators allow programmers to wrap another function in order to extend the functionality of the wrapped function, without permanently modifying it.

### Function Based

Considering the below function which basically just prints 'Hello World'.

```python
def print_greeting():
   print('Hello John')
```

The above function's behavior can be extended from just printing a one-line sentence to doing several things. So let's create our first decorator.

```python
def extend_our_function(func):
    def wrapper():
        print('function start')
        func()
        print('function end')
    return wrapper
```

The above decorator function is a regular function that takes another function as an argument, in our case - func, while in it we create another function which we use to extend the functionality of our normal function, in our case print_greeting, and then we return the newly created function - wrapper. To make use of the just created decorator we can simply do this


```python
# supply the function we want to extend its functionality as the argument to our
# decorator
print_greeting = extend_our_function(print_greeting)
# invoke the returned function we store in the variable print_greeting
print_greeting()
```

But instead of the above, python has a syntax sugar as seen below, we just add the decorator at the top of the function we want to extend its functionality using @ along with the decorator function name.


```python
@extend_our_function
def print_greeting():
   print('Hello John')
```

If our print_greeting function instead takes an argument,

```python
def print_greeting(name):
    print(f'Hello {name}')
```

then, we will modify the wrapper function in our decorator to take the name parameter, P.S we can also use - args, *kwargs

```python
def extend_our_function(func):
    def wrapper(name):
        print(f'function start')
        func(name)
        print('function end')
    return wrapper
```

A more complicated example will be if the decorator itself needs some kind of argument and also the function that is to be extended also takes some arguments.

```python
def expect_result_less_than(max_value):
    def inner_fuction(func):
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            if result > max_value:
                raise Exception(f'The value should be less than {max_value}')
            return result
        return wrapper
    return inner_fuction

@expect_result_less_than(8)
def add(a, b):
    return a + b

print(add(2,5)) # this passes
print(add(2,12)) # this fails
```

Question: print(add(2,5)) pass while print(add(2,12)) fails, WHY?

Learn More 

1. [Primer on Python Decorators](https://realpython.com/primer-on-python-decorators/)

2. [Decorators in Python](https://www.geeksforgeeks.org/decorators-in-python/#:~:text=Decorators%20are%20a%20very%20powerful,function%2C%20without%20permanently%20modifying%20it.)