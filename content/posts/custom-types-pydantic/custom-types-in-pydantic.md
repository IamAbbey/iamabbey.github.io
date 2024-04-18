---
title: 'Custom Types in Pydantic'
date: 2024-03-29T23:03:35+01:00
# weight: 1
# aliases: ["/first"]
tags: ["python", "pydantic"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
summary: "Introduction to Pydantic and Custom Types in Pydantic"
description: "Extending Pydantic with complex custom type"
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
    image: "https://repository-images.githubusercontent.com/90194616/6d31d0d9-6770-4cbc-90d5-a611662126ee" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/IamAbbey/iamabbey.github.io/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

Since software's inception, one of important aspects of software development has been data validation, it is important to validate inputs supplied into your application, so as to guarantee the data is exactly as expected.

Around 2018, Pydantic was introduced to the Python ecosystem and this data validation library receives alot of positive embrace from the community, since then Pydantic has proceeded to been the most widely used data validation library for Python, used by leading industeries and packages. [Who is using Pydantic?](https://docs.pydantic.dev/dev/#who-is-using-pydantic).

Still needs more convincing why you should use Pydantic? [Read here](https://docs.pydantic.dev/dev/why/#using-pydantic)

> All the code blocks can be copied and used directly (they are actually tested Python files).

A simple example to show Pydantic.

```python
from datetime import datetime
from typing import List, Optional
from pydantic import BaseModel


class User(BaseModel):
    id: int
    name: str = "John Doe"
    signup_ts: Optional[datetime] = None
    friends: List[int] = []


external_data = {
    "id": "123",
    "signup_ts": "2017-06-01 12:22",
    "friends": [1, "2", b"3"],
}
user = User(**external_data)
print(user)
# > User id=123 name='John Doe' signup_ts=datetime.datetime(2017, 6, 1, 12, 22) friends=[1, 2, 3]
print(user.id)
# > 123
```

As seen in the example above, Pydantic validates the supplied input `external_data` against our User model structure and ensures the supplied input conforms to our expected 'data'.

### Pydantic Types

Pydantic supports many common types from the Python standard library [Common Types](https://docs.pydantic.dev/dev/api/standard_library_types/), also it support stricter processing of this common types [Strict Types](https://docs.pydantic.dev/dev/concepts/types/#strict-types).

Pydantic also includes some custom types (e.g. to require a positive int). [Pydantic Types](https://docs.pydantic.dev/dev/api/types/).

Now to the purpose of this post, let look at how we can utilize Pydantic validation in more complex way.

Alongside the above types mentioned, you can also define your own custom data types. There are several ways to achieve it.

#### Composing types via `Annotated`
Pydantic takes advantage of `Annotated` introduced in [PEP 593](https://peps.python.org/pep-0593/) to allow us to create types that are identical to the original type as far as type checkers are concerned, but add validation, serialize differently, etc.
[read more](https://docs.pydantic.dev/dev/concepts/types/#composing-types-via-annotated)

```python {hl_lines=[15,18]}
import keyword
from typing import Annotated
from pydantic import BaseModel, AfterValidator, Field, ValidationError


def validate_name(name: str):
    if keyword.iskeyword(name):
        raise ValueError(
            f"{name} is not a valid name, please make sure it is not a python keyword."
        )
    return name


# A custom type that ensure that a string is not a keyword.
Identifier = Annotated[str, AfterValidator(validate_name)]

# A custom type `PositiveInt` to validate that supplied input is  a positive integer
PositiveInt = Annotated[int, Field(gt=0)]


class Person(BaseModel):
    name: Identifier
    age: PositiveInt


external_data = {"name": "John", "age": 18}
person = Person(**external_data)
print(person)
# > User name='John' age=18

try:
    wrong_data = {"name": "class", "age": -18}
    person = Person(**wrong_data)
    print(person)
except ValidationError as exc:
    print(exc)
    """
    2 validation errors for Person
    name
    Value error, class is not a valid name, please make sure it is not a python keyword. [type=value_error, input_value='class', input_type=str]
    
    age
    Input should be greater than 0 [type=greater_than, input_value=-18,
    """

```


#### Customizing validation with `__get_pydantic_core_schema__`
To do more extensive customization of how Pydantic handles custom classes, you can implement a special `__get_pydantic_core_schema__` to tell Pydantic how to generate the pydantic-core schema.

We will be using the `__get_pydantic_core_schema__` approach to create a more granular Pydantic custom type.

As an example we will be writing a Pydantic custom type validator called `DependsOn`; the purpose of this custom type is to have a field whose value 'depends on' on the value of another field meeting specified condition.

```python {hl_lines=[40]}
from dataclasses import dataclass
from typing import Annotated, Any, Callable
from pydantic import (
    BaseModel,
    GetCoreSchemaHandler,
    ValidationError,
    ValidationInfo,
)
from pydantic_core import core_schema
import inspect


# The frozen=True specification makes DependsOnValidator hashable.
# Without this, a union on the custom type such as X | None will raise an error.
@dataclass(frozen=True)
class DependsOnValidator:
    """Custom type that let you define a field that depends on another field."""

    depends_on: str
    depends_on_conditon: Callable[[Any], bool]
    value_conditon: Callable[[Any], bool]

    def validate(self, value, info: ValidationInfo):
        if self.depends_on not in info.data:
            raise ValueError(
                f"{info.field_name} is only allowed in model with {self.depends_on}"
            )

        if self.value_conditon(value) is True:
            if self.depends_on_conditon(info.data[self.depends_on]) is True:
                return value
            else:
                raise ValueError(
                    f"{info.field_name} is only allowed when {self.depends_on} pass condition \
                    `{inspect.getsource(self.depends_on_conditon)}`"
                )

        return value

    def __get_pydantic_core_schema__(
        self, source_type: Any, handler: GetCoreSchemaHandler
    ) -> core_schema.CoreSchema:
        return core_schema.with_info_after_validator_function(
            self.validate, handler(source_type), field_name=handler.field_name
        )


# Example using our custom validation
class Person(BaseModel):
    name: str
    age: int
    is_adult: Annotated[
        bool,
        DependsOnValidator(
            depends_on="age",
            depends_on_conditon=lambda age: age >= 18,
            value_conditon=lambda v: v is True,
        ),
    ]
    can_drink: Annotated[
        bool,
        DependsOnValidator(
            depends_on="is_adult",
            depends_on_conditon=lambda v: v is True,
            value_conditon=lambda v: v is True,
        ),
    ]


# The above schema defines that
# - `is_adult` is only allowed to be `True` if age is set and age is greater than 18
# - `can_drink` is only allowed to be `True` if `is_adult` is True

# This gives room to create a chain depends on properties `can_drink` depends on `is_adult` which in itself depends on `age`

person = Person(name="John Doe", age=18, is_adult=True, can_drink=True)  # Correct
print(person)
# > name='John Doe' age=18 is_adult=True can_drink=True

try:
    Person(
        name="John Doe", age=12, is_adult=True, can_drink=True
    )  # `is_adult` fails which inturn makes `can_drink` fail
except ValidationError as exc:
    print(exc)
    """
    2 validation errors for Person
    is_adult
    Value error, is_adult is only allowed when age pass condition `depends_on_conditon=lambda age: age >= 18,` 
    [type=value_error, input_value=True, input_type=bool]
    
    can_drink
    Value error, can_drink is only allowed in model with is_adult [type=value_error, input_value=True, input_type=bool]
    """

try:
    Person(
        name="John Doe", age=18, is_adult=False, can_drink=True
    )  # `can_drink` fails because it depends on `is_adult` to be True
except ValidationError as exc:
    print(exc)
    """
    1 validation error for Person
    can_drink
    Value error, can_drink is only allowed when is_adult pass condition `depends_on_conditon=lambda v: v is True,`
    [type=value_error, input_value=True, input_type=bool]
    """

```

#### Useful links

- [Official Pydantic documentation](https://docs.pydantic.dev/dev/)