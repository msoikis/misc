# Advanced IoC Architecture: Multi-Target Publishing & Shared Infrastructure
As your architecture grows, you inevitably encounter advanced Dependency Injection scenarios. This updated design solves three major real-world challenges:
1. **Shared Infrastructure State:** `Consumer` and `Publisher` shouldn't manage network credentials themselves. They need a shared `RabbitMQ` connection class.
2. **Circular Dependency Avoidance:** `Publisher` used to depend on `Logger`. But now `Logger` depends on `Publisher` to stream logs. If they depend on each other, that is an unresolvable infinite loop. We solve this by making the `Publisher` a pure infrastructure class, while `Logger` handles the logging semantics.
3. **The Multi-Instance Problem:** Service B needs *two* `Publisher`s. The Logger needs a `Publisher`. If everything asks for `<class 'Publisher'>`, Dishka will overwrite them! We solve this by using Python's `NewType` to create distinct Type Signatures for each unique target.
---
## 1. Updated Configuration Management (Pydantic)
We define target schemas so each publisher gets its exact routing parameters.
```python
# config.py
from pydantic import BaseModel
import yaml
class RabbitMQConfig(BaseModel):
    host: str
    user: str
    password: str
class TargetConfig(BaseModel):
    exchange: str
    routing_key: str
    queue_name: str
class CommonConfig(BaseModel):
    db_connection_string: str
    log_file_path: str
    rabbitmq: RabbitMQConfig
    log_target: TargetConfig
class ServiceAConfig(BaseModel):
    consume_queue: str
    processed_target: TargetConfig
class ServiceBConfig(BaseModel):
    consume_queue: str
    target_1: TargetConfig
    target_2: TargetConfig
```
---
## 2. Shared Classes & Strict Typing with `NewType`
Here is where we solve the "Multi-Instance" problem. By using `NewType`, we tell Pyright, PyCharm, and Dishka: *"Treat these as entirely different classes, even though structurally they are just Publishers."*
```python
# shared.py
from dataclasses import dataclass
from typing import NewType
@dataclass
class RabbitMQ:
    """Shared connection state for all RMQ components."""
    host: str
    user: str
    password: str
    
    def connect(self):
        return f"Connected to amqp://{self.user}@{self.host}"
@dataclass
class Publisher:
    """Pure infrastructure class. Does not depend on Logger."""
    rabbitmq: RabbitMQ
    exchange: str
    routing_key: str
    queue_name: str
    
    def publish(self, message: str) -> None:
        print(f"[{self.exchange}|{self.routing_key}] Published: {message}")
# --- STRICT TYPE IDENTIFIERS FOR DISHKA ---
LogPublisher = NewType("LogPublisher", Publisher)
TargetAPublisher = NewType("TargetAPublisher", Publisher)
TargetB1Publisher = NewType("TargetB1Publisher", Publisher)
TargetB2Publisher = NewType("TargetB2Publisher", Publisher)
# ------------------------------------------
@dataclass
class Logger:
    """Logger now requires its specific LogPublisher."""
    log_file_path: str
    publisher: LogPublisher 
    
    def info(self, message: str) -> None:
        # 1. Write to local file
        print(f"[FILE: {self.log_file_path}] {message}")
        # 2. Publish log over network
        self.publisher.publish(f"LOG_STREAM: {message}")
@dataclass
class Consumer:
    rabbitmq: RabbitMQ
    queue_name: str
    logger: Logger
    
    def consume(self) -> list[str]:
        self.logger.info(f"Consuming from {self.queue_name}...")
        return ["msg_alpha", "msg_beta"]
# (DbConnection, StatusDal, StateDal, UserStatus remain same as previous step)
```
---
## 3. Service Drivers
The drivers now explicitly demand their customized Publisher types, guaranteeing they never accidentally publish to the wrong exchange.
```python
# drivers.py
from dataclasses import dataclass
from shared import Consumer, Logger, StateDal, UserStatus
from shared import TargetAPublisher, TargetB1Publisher, TargetB2Publisher
@dataclass
class ServiceADriver:
    consumer: Consumer
    logger: Logger
    state_dal: StateDal
    user_status: UserStatus
    publisher: TargetAPublisher  # Strictly isolated target
    
    def run(self) -> None:
        for msg in self.consumer.consume():
            self.logger.info(f"ServiceA processing msg: {msg}")
            self.state_dal.update_state(msg)
            self.publisher.publish(f"Processed_{msg}")
@dataclass
class ServiceBDriver:
    consumer: Consumer
    logger: Logger
    user_status: UserStatus
    publisher_1: TargetB1Publisher # First distinct target
    publisher_2: TargetB2Publisher # Second distinct target
    
    def run(self) -> None:
        for msg in self.consumer.consume():
            self.logger.info(f"ServiceB splitting msg: {msg}")
            self.publisher_1.publish(f"Segment1_{msg}")
            self.publisher_2.publish(f"Segment2_{msg}")
```
---
## 4. The Dishka IoC Providers
We now wire the dependency graph. Note how `CommonProvider` manages the singleton `RabbitMQ` class, and injects it into every `Publisher` factory automatically.
```python
# ioc.py
from dishka import Provider, Scope, provide
from config import CommonConfig, ServiceAConfig, ServiceBConfig
from shared import (
    RabbitMQ, Publisher, Logger, Consumer, 
    LogPublisher, TargetAPublisher, TargetB1Publisher, TargetB2Publisher
)
from drivers import ServiceADriver, ServiceBDriver
class CommonProvider(Provider):
    def __init__(self, config: CommonConfig):
        super().__init__()
        self.config = config
    @provide(scope=Scope.APP)
    def rabbitmq(self) -> RabbitMQ:
        # Global connection shared across ALL consumers and publishers
        return RabbitMQ(
            host=self.config.rabbitmq.host,
            user=self.config.rabbitmq.user,
            password=self.config.rabbitmq.password
        )
    @provide(scope=Scope.APP)
    def log_publisher(self, rmq: RabbitMQ) -> LogPublisher:
        # Create a Publisher, but cast it as a `LogPublisher` for DI resolution
        return LogPublisher(Publisher(
            rabbitmq=rmq,
            exchange=self.config.log_target.exchange,
            routing_key=self.config.log_target.routing_key,
            queue_name=self.config.log_target.queue_name
        ))
    @provide(scope=Scope.APP)
    def logger(self, log_pub: LogPublisher) -> Logger:
        # Dishka will automatically fetch the exact LogPublisher created above
        return Logger(log_file_path=self.config.log_file_path, publisher=log_pub)
class ServiceAProvider(Provider):
    def __init__(self, config: ServiceAConfig):
        super().__init__()
        self.config = config
    @provide(scope=Scope.APP)
    def consumer(self, rmq: RabbitMQ, logger: Logger) -> Consumer:
        return Consumer(rmq, self.config.consume_queue, logger)
    @provide(scope=Scope.APP)
    def target_a_publisher(self, rmq: RabbitMQ) -> TargetAPublisher:
        # Notice we inject `rmq` which was seamlessly provided by CommonProvider!
        return TargetAPublisher(Publisher(
            rabbitmq=rmq,
            exchange=self.config.processed_target.exchange,
            routing_key=self.config.processed_target.routing_key,
            queue_name=self.config.processed_target.queue_name
        ))
    @provide(scope=Scope.APP)
    def driver(
        self, consumer: Consumer, logger: Logger, 
        state_dal: StateDal, user_status: UserStatus,
        publisher: TargetAPublisher
    ) -> ServiceADriver:
        return ServiceADriver(consumer, logger, state_dal, user_status, publisher)
class ServiceBProvider(Provider):
    def __init__(self, config: ServiceBConfig):
        super().__init__()
        self.config = config
    # ... consumer definition omitted for brevity ...
    @provide(scope=Scope.APP)
    def publisher_1(self, rmq: RabbitMQ) -> TargetB1Publisher:
        return TargetB1Publisher(Publisher(
            rabbitmq=rmq, 
            exchange=self.config.target_1.exchange,
            routing_key=self.config.target_1.routing_key,
            queue_name=self.config.target_1.queue_name
        ))
    @provide(scope=Scope.APP)
    def publisher_2(self, rmq: RabbitMQ) -> TargetB2Publisher:
        return TargetB2Publisher(Publisher(
            rabbitmq=rmq, 
            exchange=self.config.target_2.exchange,
            routing_key=self.config.target_2.routing_key,
            queue_name=self.config.target_2.queue_name
        ))
    @provide(scope=Scope.APP)
    def driver(
        self, consumer: Consumer, logger: Logger, user_status: UserStatus,
        publisher_1: TargetB1Publisher, publisher_2: TargetB2Publisher
    ) -> ServiceBDriver:
        return ServiceBDriver(consumer, logger, user_status, publisher_1, publisher_2)
```
## Summary of Architectural Upgrades
1. **DAG Integrity (Directed Acyclic Graph):** We broke the circular dependency (`Logger` -> `Publisher` -> `Logger`) by removing the Logger dependency from the Publisher. Now, infrastructure connections sit at the true bottom of the dependency graph.
2. **`NewType` Isolation:** We created `TargetB1Publisher` and `TargetB2Publisher`. Dishka resolves them as entirely separate dependencies, allowing Service B to request both simultaneously without conflicts. 
3. **Implicit Cross-Provider Injection:** `ServiceBProvider` demands `RabbitMQ` to build its publishers. It doesn't configure it. Because `CommonProvider` is loaded alongside it at startup, Dishka grabs the shared `Scope.APP` Singleton `RabbitMQ` connection from Common, and passes it into Service B.
