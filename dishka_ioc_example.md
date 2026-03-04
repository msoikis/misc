# Modern Python IoC Architecture: Dishka + Pydantic
Based on your strict requirements for **type safety, PyCharm autocomplete**, and compliance with strict static analyzers like **basedpyright** and **ruff**, the absolute best modern IoC framework is **[Dishka](https://dishka.readthedocs.io/)**.
## Why Dishka?
Historically, Python DI frameworks (like `dependency-injector`) relied heavily on magic methods, dynamic attribute resolution (`**kwargs`), or string-based injection. This breaks PyCharm's inference and causes Pyright rules to fail constantly. 
**Dishka** was built from the ground up for modern Python:
1. **Zero Magic:** It relies purely on standard Python type hints (`__init__`).
2. **Type-Safe Containers:** Calling `container.get(DbConnection)` statically resolves to `DbConnection`. Your IDE natively understands it.
3. **No Framework Pollution:** Your domain classes don't need any `@inject` decorators. They remain 100% independent.
4. **Explicit Scoping:** Clearly separates out `APP` (Singletons) vs `REQUEST` (per-call instances).
### How IoC and DI Work (and the Benefits)
- **Dependency Injection (DI)** means a class receives its dependencies from the outside rather than creating them itself (e.g., passing a `Logger` into `__init__` rather than calling `Logger()` inside the class).
- **Inversion of Control (IoC)** is having a centralized "Container" manage the lifecycle and wiring of all these dependencies.
- **Benefits:** 
  - **Decoupling:** `ServiceA` doesn't care how `RabbitMQ` is configured, it just uses the `Consumer`.
  - **Single Responsibility:** Classes focus on business logic. The DI Container handles initialization logic.
  - **Tremendous Testability:** We can effortlessly swap out a real `Publisher` for a `MockPublisher` during unit tests without modifying the business code.
---
## 1. Configuration Management
We use **Pydantic** to parse the YAML config files. This guarantees our configuration is strongly typed, giving you auto-complete for config values and strict validation during startup.
```python
# config.py
from pydantic import BaseModel
import yaml
from pathlib import Path
# --- Configuration Models ---
# By using Pydantic, basedpyright knows exactly what fields exist.
class CommonConfig(BaseModel):
    db_connection_string: str
    log_file_path: str
    rabbitmq_config: dict
class ServiceConfig(BaseModel):
    queue_consume: str
    queue_publish: str
# Helper to load yaml into our type-safe models
def load_config(file_path: str, model: type[BaseModel]) -> BaseModel:
    with open(file_path, 'r') as f:
        data = yaml.safe_load(f)
    return model(**data)
```
## 2. Shared Domain & Infrastructure Classes
Notice how **none of these classes import Dishka**. They are just standard Python classes. This is clean architecture at its finest.
```python
# shared.py
import logging
from dataclasses import dataclass
@dataclass
class DbConnection:
    """Encapsulates the DB connection."""
    connection_string: str
    
    def connect(self):
        return f"Connected to {self.connection_string}"
@dataclass
class Logger:
    """Writes to a log file."""
    log_file_path: str
    
    def info(self, message: str) -> None:
        print(f"[INFO - {self.log_file_path}] {message}")
@dataclass
class Consumer:
    """Encapsulates RabbitMQ consume functionality."""
    logger: Logger
    queue_name: str
    
    def consume(self) -> list[str]:
        self.logger.info(f"Consuming from {self.queue_name}...")
        return ["msg1", "msg2"]
@dataclass
class Publisher:
    """Encapsulates RabbitMQ publish functionality."""
    logger: Logger
    queue_name: str
    
    def publish(self, message: str) -> None:
        self.logger.info(f"Published '{message}' to {self.queue_name}")
@dataclass
class StatusDal:
    """Data Access Layer for Status Table."""
    db: DbConnection
    
    def save_status(self, status: str) -> None:
        pass # interaction with self.db
@dataclass
class StateDal:
    """Data Access Layer for State Table."""
    db: DbConnection
    
    def update_state(self, message: str) -> None:
        pass # interaction with self.db
@dataclass
class UserStatus:
    """Handles business logic for sending and updating user status."""
    logger: Logger
    status_dal: StatusDal
    publisher: Publisher
    
    def update_status(self, user_name: str, new_status: str) -> None:
        # 1. Log the update
        self.logger.info(f"Updating status for {user_name} to {new_status}")
        # 2. Persist to DB
        self.status_dal.save_status(new_status)
        # 3. Publish to RMQ
        self.publisher.publish(f"StatusUpdate({user_name}, {new_status})")
```
## 3. Service Drivers
The drivers implement the core execution paths. They declare what they need to function.
```python
# drivers.py
from shared import Consumer, Logger, StateDal, UserStatus, Publisher
from dataclasses import dataclass
@dataclass
class ServiceADriver:
    """
    Service A Driver Business Logic.
    Consumes MSGs -> Updates State in DB -> Calls UserStatus -> Publishes processed.
    """
    consumer: Consumer
    logger: Logger
    state_dal: StateDal
    user_status: UserStatus
    publisher: Publisher
    
    def run(self) -> None:
        messages = self.consumer.consume()
        for msg in messages:
            self.logger.info(f"ServiceA processing msg: {msg}")
            self.state_dal.update_state(msg)
            self.user_status.update_status(user_name="UserA", new_status="Processed")
            self.publisher.publish(f"Processed_{msg}")
@dataclass
class ServiceBDriver:
    """
    Service B Driver Business Logic.
    Consumes MSGs -> Logs updates -> Calls UserStatus -> Publishes result.
    """
    consumer: Consumer
    logger: Logger
    user_status: UserStatus
    publisher: Publisher
    
    def run(self) -> None:
        messages = self.consumer.consume()
        for msg in messages:
            self.logger.info(f"ServiceB logging update for msg: {msg}")
            self.user_status.update_status(user_name="UserB", new_status="Finalized")
            self.publisher.publish(f"Result_{msg}")
```
## 4. The IoC Framework Setup (Dishka)
Here is where the magic happens. We configure the dependencies centrally using Dishka `Providers`. 
- `@provide` marks a function as the factory for a specific type type.
- Basedpyright and PyCharm analyze this effortlessly because they are simple python functions returning types.
```python
# ioc.py
from dishka import Provider, Scope, provide, make_container
from shared import DbConnection, Logger, Consumer, Publisher, StatusDal, StateDal, UserStatus
from drivers import ServiceADriver, ServiceBDriver
from config import CommonConfig, ServiceConfig
class CommonProvider(Provider):
    """Provides shared singletons (APP scope means Singleton)"""
    
    def __init__(self, common_config: CommonConfig):
        super().__init__()
        self.common_config = common_config
    @provide(scope=Scope.APP)
    def db_connection(self) -> DbConnection:
        # We manually instantiate objects inside the provider, injecting our config!
        return DbConnection(connection_string=self.common_config.db_connection_string)
    @provide(scope=Scope.APP)
    def logger(self) -> Logger:
        return Logger(log_file_path=self.common_config.log_file_path)
    # Dishka automatically passes `db` into `status_dal` by inspecting the type hint!
    @provide(scope=Scope.APP)
    def status_dal(self, db: DbConnection) -> StatusDal:
        return StatusDal(db=db)
    @provide(scope=Scope.APP)
    def state_dal(self, db: DbConnection) -> StateDal:
        return StateDal(db=db)
    # Dishka figures out the dependency graph magically.
    @provide(scope=Scope.APP)
    def user_status(self, logger: Logger, status_dal: StatusDal, publisher: Publisher) -> UserStatus:
        return UserStatus(logger, status_dal, publisher)
class ServiceAProvider(Provider):
    """Provides the dependencies specifically scoped for Service A context"""
    
    def __init__(self, service_config: ServiceConfig):
        super().__init__()
        self.config = service_config
    @provide(scope=Scope.APP)
    def consumer(self, logger: Logger) -> Consumer:
        return Consumer(logger=logger, queue_name=self.config.queue_consume)
    @provide(scope=Scope.APP)
    def publisher(self, logger: Logger) -> Publisher:
        return Publisher(logger=logger, queue_name=self.config.queue_publish)
    @provide(scope=Scope.APP)
    def driver(self, consumer: Consumer, logger: Logger, state_dal: StateDal, 
               user_status: UserStatus, publisher: Publisher) -> ServiceADriver:
        return ServiceADriver(consumer, logger, state_dal, user_status, publisher)
# (You would have a similar ServiceBProvider class for Service B)
```
## 5. Bootstrapping and Execution
When running the real app, we load the configs and compose the containers. 
```python
# main_a.py
from config import CommonConfig, ServiceConfig, load_config
from ioc import CommonProvider, ServiceAProvider
from dishka import make_container
from drivers import ServiceADriver
def run_service_a():
    # 1. Load Configurations
    common_cfg = load_config("common_config.yaml", CommonConfig)
    service_a_cfg = load_config("service_a_config.yaml", ServiceConfig)
    # 2. Wire up the IoC Container using the Providers
    container = make_container(
        CommonProvider(common_cfg),
        ServiceAProvider(service_a_cfg)
    )
    # 3. Retrieve the fully built Driver. 
    # PyCharm and Pyright knows `driver` is an instance of `ServiceADriver` exactly!
    driver = container.get(ServiceADriver)
    
    # 4. Start the app lifecycle
    driver.run()
if __name__ == "__main__":
    run_service_a()
```
## 6. Effortless Unit / Integration Testing
Testing is trivialized because of DI. You don't need `unittest.mock.patch` which is string-based and breaks during refactoring. Instead, we can create a `TestProvider` that specifically overrides infrastructure classes with fully typed stubs!
```python
# tests/test_service_a.py
import pytest
from dishka import Provider, Scope, provide, make_container
from drivers import ServiceADriver
from shared import Consumer, Publisher
from config import CommonConfig, ServiceConfig
from ioc import CommonProvider, ServiceAProvider
class MockPublisher(Publisher):
    """A type-safe mock publisher that captures messages in memory"""
    published_messages = []
    
    def publish(self, message: str) -> None:
        self.published_messages.append(message)
class OverrideProvider(Provider):
    """This provider overrides explicit bindings for testing."""
    
    @provide(scope=Scope.APP)
    def publisher(self) -> Publisher:
        # Overrides the Publisher provided in ServiceAProvider
        # Pyright loves this because MockPublisher is a subclass of Publisher.
        return MockPublisher(logger=None, queue_name="mock_queue")
def test_service_a_driver_publishes_correctly():
    # Setup configs
    common_cfg = CommonConfig(db_connection_string="test_db", log_file_path="null", rabbitmq_config={})
    svc_cfg = ServiceConfig(queue_consume="test_in", queue_publish="test_out")
    
    # Build a Test Container! 
    # Dishka layers providers. The later ones override the former ones.
    container = make_container(
        CommonProvider(common_cfg),
        ServiceAProvider(svc_cfg),
        OverrideProvider() # Injects our mock overrides
    )
    # Act
    driver = container.get(ServiceADriver)
    driver.run()
    
    # Assert
    mock_pub = container.get(Publisher)
    assert len(mock_pub.published_messages) > 0
    assert "Processed_msg1" in mock_pub.published_messages
```
### Recap of the Flow & Types
If using PyCharm or checking with Ruff/BasedPyright:
- Using `dataclasses` forces you to be explicit about types.
- Loading YAML via Pydantic catches missing config keys before your app even tries to start.
- `Dishka` Providers map concrete classes into the dependency tree while providing 100% type hinting correctness across the board. 
- Mocks are real inherited Python objects, eliminating the fragile string references required by `unittest.mock.patch()`.
