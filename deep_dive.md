# Dishka & Python IoC: Deep Dive 
Here is an advanced explanation addressing all the inner mechanics of Dishka, Python type checking, and best practices for building enterprise-grade architectures.
---
## 1. Class Definitions: Pydantic vs Dataclasses vs Standard classes
You can use **any of the three**, but they serve fundamentally different purposes in a well-architected application. Dishka does not care what you use, because it only looks at Python Type definitions. 
### My Recommendation:
- **Use Pydantic for the Boundaries** (Config files, HTTP JSON Requests, Database Models). Pydantic is a powerful *Validation* library. It does a lot of heavy lifting under the hood to cast types and enforce rules. 
- **Use `dataclasses` (or standard `__init__` classes) for Domain/Services** (Like `Consumer`, `ServiceDriver`). They are pure, blazing fast, and carry zero overhead or third-party behaviors. 
- **Standard `__init__` vs `dataclass`?** They are essentially identical. A `@dataclass` acts as an automated macro that writes standard `__init__` methods for you. It saves you from writing `self.foo = foo` repeatedly.
**Example Comparison:**
```python
# 1. Standard Python Class (Great, but verbose)
class Logger:
    def __init__(self, log_path: str):
        self.log_path = log_path
# 2. Dataclass (Identical to above, strictly recommended for Services)
from dataclasses import dataclass
@dataclass
class Logger:
    log_path: str
# 3. Pydantic (NOT recommended for internal services, adds overhead)
from pydantic import BaseModel
class Logger(BaseModel):
    log_path: str
```
---
## 2. Understanding `@provide(scope=Scope.APP)`
### What does "Scope" mean?
A Scope defines the **lifecycle** (lifespan) of an object. It answers the question: *"When someone asks for this object, do I give them the exact same existing instance, or do I run the factory function again to create a new one?"*
### How Dishka handles Scopes:
Dishka builds a hierarchical cache of objects.
1. **`Scope.APP` (Singleton)**: This represents the lifespan of the entire application. 
   - **How it works:** The *very first time* any part of the code asks for a `Logger`, Dishka runs your provider function. It then saves that `Logger` instance in a memory cache attached to the App Container. Every subsequent request will receive that exact same memory address. 
   - **Why you need it:** To prevent initializing heavy connections repeatedly. You want exactly *one* Database Connection Pool, and *one* global Logger per application.
2. **`Scope.REQUEST` (Per-Call/Session)**: This represents the lifespan of a single operation (e.g., handling one RabbitMQ message, or one HTTP Request). 
   - **How it works:** When a new request comes in, you create a "Sub-Container" using `with container() as request_container:`. Anything scoped to `REQUEST` will be instantiated once for that specific sub-container, and destroyed when the `with` block ends.
   - **Why you need it:** Crucial for Database Transactions (`session.commit()` -> `session.close()`). You want a new database transaction per incoming RabbitMQ message.
---
## 3. How Dishka resolves Dependencies (and Overrides/Conflicts)
### Q: How does Discord know to pass `DbConnection`? 
Dishka uses standard Python introspection (the `inspect` module) at startup. It looks at the signature of your provider:
```python
@provide(scope=Scope.APP)
def status_dal(self, db: DbConnection) -> StatusDal:
```
Dishka parses the type hint: `"This function requires the type <class 'DbConnection'> to execute"`. It then searches its internal lookup dictionary (built when you called `make_container`) for a matching return type. 
### Q: What if two providers return `DbConnection`?
Dishka resolves dependencies strictly by **Return Type**, not by the name of the function. 
If you provide multiple functions that return `DbConnection`, **the last one registered wins**. 
```python
container = make_container(
    ProviderA(),  # Returns DbConnection mapped to local DB
    ProviderB()   # Returns DbConnection mapped to cloud DB
)
```
Because `ProviderB` came last in the list, it completely overwrites `ProviderA`'s definition in the internal registry. Every service will now use the Cloud DB. This behavior is deliberate and is precisely what allows us to inject Mock classes during testing.
### Q: How does ServiceA's Provider find the CommonProvider's Logger? 
Because Providers are completely flattened and merged when you call `make_container(CommonProvider(), ServiceAProvider())`. 
- `ServiceAProvider` has no concept of `CommonProvider`. 
- `ServiceAProvider` just says to the unified Central Container: *"I need `<class 'Logger'>`"*. 
- The Central Container checks its global manifest, sees that `CommonProvider` taught it how to build a `<class 'Logger'>`, and supplies it. 
---
## 4. How PyCharm, Pyright, and `container.get(...)` work
When you type:
```python
driver = container.get(ServiceADriver)
driver.r... # PyCharm suggests driver.run()
```
You asked: **How does it know the type?** 
This is not Dishka-specific magic; this is natively built into the Python typing system using **Generics (`TypeVar`)**.
Inside Dishka's source code, the `Container` class is defined roughly like this:
```python
from typing import TypeVar
# 1. Define a Generic Variable "T"
T = TypeVar('T')
class Container:
    
    # 2. Declare that whatever type is passed into the argument, 
    # the function will return an instance of that EXACT same type.
    def get(self, dependency_type: type[T]) -> T:
        # (Internal Dishka logic resolving the instance)
        return instance
```
**What the IDE sees:**
1. You pass `ServiceADriver` (the class type itself) as the argument. 
2. PyCharm/Pyright sees `dependency_type: type[T]`. It binds `T = ServiceADriver`.
3. It then looks at the return annotation `-> T`. 
4. It dynamically replaces `T` with `ServiceADriver`. 
Thus, the static analyzer proves—without running any code—that `driver` is guaranteed to be an instance of `ServiceADriver`. You get 100% accurate autocomplete, and if you misspell a method on `driver`, `ruff` or `basedpyright` will instantly highlight it in red.
