### Single Responsibility Principle

## Table of Contents

- [Single Responsibility Principle](#single-responsibility-principle)
- [Refactoring towards the SRP](#refactoring-towards-the-srp)
- [Refactor for clarity](#refactor-for-clarity)
- [read_trade_data()](#read_trade_data)
- [parse_trades](#parse_trades)
- [validate_trade_data()](#validate_trade_data)
- [create_trade_record()](#create_trade_record)
- [store_trades()](#store_trades)
- [Refactoring for adaptability](#refactoring-for-adaptability)


---

The single responsibility principle (SRP) instructs developers to write code that has one and only one
reason to change. If a class has more than one reason to change, it has more than one responsibility.
Classes with more than a single responsibility should be broken down into smaller classes, each of
which should have only one responsibility and reason to change.

It is difficult to overstate the importance of delegating to abstractions. It is the lynchpin of adaptive
code and, without it, developers would struggle to adapt to changing requirements in the way
that Scrum and other Agile processes demand.

Let us meet Vincent. Vincent is a developer and he loves his job really a lot. Vincent loves to keep learning and he buys books that talk about software but he is always busy that he fails to read them. Vincent has a new client that wants an application developed for him.

**_The client wants a program that reads trade records from a file, parse them, log any errors, process the records and them save them to a database._**

The data is stored in the following format. The first 3 capitals are the source currency code, the next 3 capitals are the destination currency code. The first integer is the lot and the last float is the price.

```markdown
UGAUSD,2,45.3
UGAUSD,7,76.4
UGAEUR,7,76.4
HJDSGS,1,76.3
ygfuhf,tj,89
```

With these requirements, Vincent works out a first prototype of this application and tests to see if it works as the client wanted. Below is the class code.

**Initial TradeProcessor structure:** one public method, `process_trades()`, processes the trade records.
All reading, parsing, validation, logging and persistence are hidden inside that single operation.

```python
from typing import List
from sqlalchemy import create_engine, Column, Integer, String, Float
from sqlalchemy.orm import sessionmaker
from base import Base

class TradeProcessor(object):
    @staticmethod
    def process_trades(filename):
        lines: List[str] = []
        with open(filename) as ft:
            for line in ft: lines.append(line)
        trades: List[TradeRecord] = []

        for index, line in enumerate(lines):
            fields = line.split(',')
            if len(fields) != 3:
                print(f'Line {index} malformed. Only {len(fields)} field(s) found.')
                continue
            if len(fields[0]) != 6:
                print(f'Trade currencies on line {index} malformed: "{fields[0]}"')
                continue
            trade_amount = 0
            try:
                trade_amount = float(fields[1])
            except ValueError:
                print(f"WARN: Trade amount on line {index} not a valid integer: '{fields[1]}'")

            trade_price = 0
            try:
                trade_price = float(fields[2])
            except ValueError:
                print(f"WARN: Trade price on line {index} not a valid decimal:'{fields[2]}'")

            print(trade_amount)
            sourceCurrencyCode = fields[0][:3]
            destinationCurrencyCode = fields[0][3:]
            trade = TradeRecord(source=sourceCurrencyCode, dest=destinationCurrencyCode,
                                lots=trade_amount, amount=trade_price)
            trades.append(trade)

        engine = create_engine('postgresql://postgres:u2402/598@localhost:5432/python')
        Session = sessionmaker(bind=engine)
        Base.metadata.create_all(engine)
        session = Session()
        for trade in trades:
            session.add(trade)
        session.commit()
        session.close()
```

> **Note :** In this example we used the SqlAlchemy ORM for persistence but we could have used any DB APIs out there.

Below is the code for the TradeRecord class that SqlAlchemy uses to persist our data.

```python
class TradeRecord(Base):
    __tablename__ = 'TradeRecord'
    id = Column(Integer, primary_key=True)
    source_curreny = Column(String)
    dest_currency = Column(String)
    lots = Column(Integer)
    amount = Column(Float)

    def __init__(self, source, dest, lots, amount):
        self.source_curreny = source
        self.dest_currency = dest
        self.lots = lots
        self.amount = amount
```

If you look closely at the TradeProcessor class, it is the best example of a class that has a ton of responsibilities to change. The method `process_trades` is a hidden class within itself. It is doing more than one thing as listed below.

1. It reads every line from a File object, storing each line in a list of strings.
2. It parses out individual fields from each line and stores them in a more structured list of
   Trade­Record instances.
3. The parsing includes some validation and some logging to the console.
4. Each TradeRecord is then stored to a database.

We can see that the responsibilities of the TradeProcessor are :

1. Reading files
2. Parsing strings
3. Validating string fields
4. Logging
5. Database insertion.

The single responsibility principle states that this class, like all others,
should only have a single reason to change. However, the reality of the TradeProcessor is that it will
change under the following circumstances:

- When the client decides not to use a file for input but instead read the trades from a remote call to a web service.
- When the format of the input data changes, perhaps with the addition of an extra field indicating the broker for the transaction.
- When the validation rules of the input data change.
- When the way in which you log warnings, errors, and information changes. If you are using a
  hosted web service, writing to the console would not be a viable option.
- When the database changes in some way for example the client decides not to store the data in a relational database and opt for document storage, or the database is moved behind a web service that
  you must call.

For each of these changes, this class would have to be modified. Furthermore, unless you maintain
a variety of versions, there is no possibility of adapting the TradeProcessor so that it is able to read
from a different input source, for example. Imagine the maintenance headache when you are asked to
add the ability to store the trades in a web service!!!

#### **Refactoring towards the SRP**

---

We are going to achieve this in two steps.

1. Refactor for Clarity
2. Refactor for Adaptability

#### **Refactor for clarity**

---

The first thing we are going to do is to break down the monstrous `process_trades()` method into smaller more specialized methods that do only one thing. Here we go.
If you look closely, the `process_trades()` method is doing 3 things:

1. Reading data from the file.
2. Parsing and Logging and
3. Storing to the data.

The original `process_trades()` implementation above can be divided into three consecutive blocks:

| Block | Existing code and responsibility | Extracted method |
| --- | --- | --- |
| Read and initialise | Create `lines: List[str]`, open `filename`, append each line, and initialise `trades: List[TradeRecord]`. | `__read_trade_data(filename)` reads the lines; the trades list belongs to parsing. |
| Parse, validate and log | Enumerate the lines, split each on `','`, reject rows without three fields or a six-character currency pair, attempt numeric conversion of amount and price, print warnings and the amount, split the currency pair into two three-character codes, create a `TradeRecord`, and append it to `trades`. | `__parse_trades(lines)` |
| Store | Create the SQLAlchemy engine and session factory, create the database tables, open a session, add each trade, commit, and close the session. | `__store_trades(trades)` |

The original numeric-conversion handlers print warnings but do not skip the malformed trade; this
breakdown describes the existing code before the later validation refactor.

So we can see from a very high level refactor it to something like below

```python
 @staticmethod
    def process_trades(filename):
        lines: List[str] = TradeProcessor.__read_trade_data(filename)
        trades: List[TradeRecord] = TradeProcessor.__parse_trades(lines)
        TradeProcessor.__store_trades(trades)
```

> Notice how these 4 smaller methods are easier to test than the original monolith!!

**TradeProcessor after the first extraction**

| Visibility | UML signature | Responsibility |
| --- | --- | --- |
| Public | `process_trades(String): List[TradeRecord]` | Control the entire workflow. |
| Private | `read_trade_data(String): List[str]` | Read trade data from the file. |
| Private | `parse_trades(List[str]): List[TradeRecord]` | Parse the trade data read from the file. |
| Private | `store_records(List[TradeRecord]): void` | Store the parsed data in the database. |

The Python snippets name the storage helper `__store_trades`; the UML calls it `store_records`.
The UML also labels `process_trades` with a list return type, whereas the implementation only coordinates
the workflow and returns no value.

Now let us look into the implementations of these new more focused methods.

#### read_trade_data()

---

```python
@staticmethod
    def __read_trade_data(filename: str) -> List[str]:
        lines: List[str]
        lines = [line for line in open(filename)]
        return lines
```

This method takes in the name of the file to read, it uses a list comprehension to enumerate over the read lines and returns a list of strings. Really simple!!!.

#### parse_trades

---

```python
@staticmethod
    def __parse_trades(trade_data: List[str]) -> List[TradeRecord]:
        trades: List[TradeRecord] = []
        for index, line in enumerate(trade_data):
            fields: List[str] = line.split(',')
            if not TradeProcessor.__validate_trade_data(fields, index + 1):
                continue
            trade = TradeProcessor.__create_trade_record(fields)
            trades.append(trade)
        return trades
```

This method takes in a list of strings produced by the `read_trade_data()` methods and tries to parse according to a given structure. **methods should do only one thing** and hence the `parse_trades()` method delegates to two other methods to accomplish its task.

1. The `validate_trade_data()` method. This is responsible for validating the read string to check if it follows a given format.
2. The `create_trade_record()` method. This takes in a list of validated strings and uses them to create a `TradeRecord` object to persist to the database.

Let us work on the implements of these two new methods.

#### validate_trade_data()

---

```python
   @staticmethod
    def __validate_trade_data(fields: List[str], index: int) -> bool:
        if len(fields) != 3:
            TradeProcessor.__log_message(f'WARN: Line {index} malformed. Only {len(fields)} field(s) found.')
            return False
        if len(fields[0]) != 6:
            TradeProcessor.__log_message(f'WARN: Trade currencies on line {index} malformed: {fields[0]}')
            return False
        try:
            trade_amount = float(fields[1])
        except ValueError:
            TradeProcessor.__log_message(f"WARN: Trade amount on line {index} not a valid integer: '{fields[1]}'")
            return False
        try:
            trade_price = float(fields[2])
        except ValueError:
            TradeProcessor.__log_message(f'WARN: Trade price on line {index} not a valid decimal:{fields[2]}')
            return False
        return True
```

This method should be self explanatory since it is a refactor from the original `process_trades()` method. One thing has changed in it. The method no longer does the logging by itself. It delegates the logging to another method called `log_message()`. We shall see the advantage of this later.

Below is the implementation of the `log_message()` method.

```python
@staticmethod
def __log_message(message: str) -> None:
    print(message)
```

#### create_trade_record()

---

```python
@staticmethod
    def __create_trade_record(fields: List[str]) -> TradeRecord:
        in_curr = slice(0, 3);
        out_curr = slice(3, None)
        source_curr_code = fields[0][in_curr]
        dest_curr_code = fields[0][out_curr]
        trade_amount = int(fields[1])
        trade_price = float(fields[2])

        trade_record = TradeRecord(source_curr_code, dest_curr_code,
                                   trade_amount, trade_price)
        return trade_record
```

This is also straight forward. The reason why we use slice objects here is to make our code readable. The last method we will look at is the `store_trades()` which persists our data to a database.

#### store_trades()

---

```python
@staticmethod
    def __store_trades(trades: List[TradeRecord]) -> None:
        engine = create_engine('postgresql://postgres:54875/501@localhost:5432/python')
        Session = sessionmaker(bind=engine)
        Base.metadata.create_all(engine)
        session = Session()
        for trade in trades:
            session.add(trade)
        session.commit()
        session.close()
        TradeProcessor.__log_message(f'{len(trades)} trades processed')

```

This method uses an ORM known as SQLAlchemy to persist our data. ORMs write the SQL for us behind the scene and this increases the flexibility of our application.

> This method is far from ideal, notice that it hard codes the connection strings and this very bad. There are tones of github repositories with exposed database connection strings. It would be better to read the connection string from a configuration file and add the configure file to gitignore.

At the moment, our class that had only one big method now has a bunch of methods as shown in the following code snippet and class summary:

```python

class TradeProcessor(object):
    @staticmethod
    def process_trades(filename):
        lines: List[str] = TradeProcessor.read_trade_data(filename)
        trades: List[TradeRecord] = TradeProcessor.parse_trades(lines)
        TradeProcessor.store_trades(trades)

    @staticmethod
    def __read_trade_data(filename: str) -> List[str]:
        lines: List[str]
        lines = [line for line in open(filename)]
        return lines

    @staticmethod
    def __log_message(message: str) -> None:
        print(message)

    @staticmethod
    def __validate_trade_data(fields: List[str], index: int) -> bool:
        if len(fields) != 3:
            TradeProcessor.log_message(f'Line {index} malformed. Only {len(fields)} field(s) found.')
            return False
        if len(fields[0]) != 6:
            TradeProcessor.log_message(f'Trade currencies on line {index} malformed: {fields[0]}')
            return False
        try:
            trade_amount = float(fields[1])
        except ValueError:
            TradeProcessor.log_message(f"Trade amount on line {index} not a valid integer: '{fields[1]}'")
            return False
        try:
            trade_price = float(fields[2])
        except ValueError:
            TradeProcessor.log_message(f'Trade price on line {index} not a valid decimal:{fields[2]}')
            return False
        return True

    @staticmethod
    def __create_trade_record(fields: List[str]) -> TradeRecord:
        in_curr = slice(0, 3);
        out_curr = slice(3, None)
        source_curr_code = fields[0][in_curr]
        dest_curr_code = fields[0][out_curr]
        trade_amount = int(fields[1])
        trade_price = float(fields[2])

        trade_record = TradeRecord(source_curr_code, dest_curr_code,
                                   trade_amount, trade_price)
        return trade_record

    @staticmethod
    def __parse_trades(trade_data: List[str]) -> List[TradeRecord]:
        trades: List[TradeRecord] = []
        for index, line in enumerate(trade_data):
            fields: List[str] = line.split(',')
            if not TradeProcessor.validate_trade_data(fields, index + 1):
                continue
            trade = TradeProcessor.create_trade_record(fields)
            trades.append(trade)
        return trades

    @staticmethod
    def __store_trades(trades: List[TradeRecord]) -> None:
        engine = create_engine('postgresql://postgres:u2402/501@localhost:5432/python')
        Session = sessionmaker(bind=engine)
        Base.metadata.create_all(engine)
        session = Session()
        for trade in trades:
            session.add(trade)
        session.commit()
        session.close()
        TradeProcessor.log_message(f'{len(trades)} trades processed')


```

**TradeProcessor after refactoring for clarity**

The class now has seven methods. Six are private so callers cannot invoke the individual steps out of
order; the public `process_trades` method controls their sequence.

| Visibility | UML signature | Responsibility |
| --- | --- | --- |
| Private | `create_trade_record(): void` | Map parsed fields into a trade record. |
| Private | `log_message(String): void` | Report a message. |
| Private | `parse_trades(List[str]): List[TradeRecord]` | Validate and map input lines into records. |
| Public | `process_trades(String): List[TradeRecord]` | Coordinate the workflow. |
| Private | `read_trade_data(String): List[str]` | Read input lines. |
| Private | `store_records(List[TradeRecord]): void` | Persist records. |
| Private | `validate_trade_data(int): Boolean` | Check a row's fields and report errors using its line number. |

The original UML abbreviates or differs from the Python signatures: `__create_trade_record` takes
`List[str]` and returns `TradeRecord`, `__validate_trade_data` takes both the fields and an index,
`process_trades` returns no value, and the storage helper is named `__store_trades`.

Looking back at this refactor, it is a clear improvement on the original implementation. However, what have you really achieved? Although the new ProcessTrades method is indisputably smaller
than the monolithic original, and the code is definitely more readable, you have gained very little by way of adaptability. You can change the implementation of the LogMessage method so that it, for example, writes to a file instead of to the console, but that involves a change to the TradeProcessor class, which is precisely what you wanted to avoid.

This refactor has been an important stepping stone on the path to truly separating the responsibilities of this class. It has been a **refactor for clarity**, not for adaptability. The next task is to split each responsibility into different classes and place them behind interfaces. What you need is true abstraction to achieve useful adaptability.

#### **Refactoring for adaptability**

---

In the previous refactor, we broke down the `process_trades()` method into smaller more focused methods. But still, that didn't solve our problem, our class was still doing lots of things. In this section, we are going to distribute the different responsibilities across classes.

From the previous section, we agreed that our class was serving 3 main responsibilities, Data reading, Data parsing and data storage. So we will start with taking out the code that does that into other classes.

We are going to create 3 abstract classes that will be used by the TradeProcessor class as described below.

**TradeProcessor and its three dependencies**

`TradeProcessor` exposes `process_trades(): void` and holds three private collaborators:

| Private field | Interface | Public interface operation | Responsibility |
| --- | --- | --- | --- |
| `trade_data_provider` | `DataProvider` | `read_trade_data(): List[str]` | Supply input lines. |
| `trade_data_parser` | `TradeDataParser` | `parse_trade_data(List[str]): List[TradeRecord]` | Convert input lines into records. |
| `trade_data_persister` | `TradeRepository` | `persist_trade_data(List[TradeRecord]): void` | Store the records. |

These are dependencies held by `TradeProcessor`, not parent classes it inherits from. The responsibilities
are now distributed among separate abstract classes. The original UML misspells the provider operation
as `read_trade_date()`; the table uses `read_trade_data()` to match the code.

The TradeProcessor class now has private polymorphic hidden fields that it uses to accomplish its tasks. Since we already created smaller specific methods, we know which method goes to which abstraction. Below are the implementations of the new abstract classes.

```python
class DataProvider(ABC):
    @abstractmethod
    def read_trade_data(self):
        pass


class TradeDataParser(ABC):
    @abstractmethod
    def parse_trade_data(self, lines: List[str]) -> List[TradeRecord]:
        pass


class TradeRepository(ABC):
    @abstractmethod
    def persist_trade_data(self, trade_data: List[TradeRecord]) -> None:
        pass
```

Notice that all of them are abstract classes with abstract methods and so can't be directly instantiated. We shall then have implementors of these abstract classes to use with the `TradeProcess()` class.

Below is the new implementation of the TradeProcessor class.

```python
class TradeProcessor(object):
    def __init__(self, provider: DataProvider, parser: TradeDataParser,
                 persister: TradeRepository) -> None:
        self._provider = provider
        self._parser = parser
        self._persister = persister

    def process_trades(self):
        lines = self._provider.read_trade_data()
        trades = self._parser.parse_trade_data(lines)
        self._persister.persist_trade_data(trades)
```

We are now doing it the object oriented way, we are having objects encapsulating computations (wait for the **strategy pattern** later). The objects that do the real work are injected into the TradeProcessor class when it is being instantiated. This is an example of **dependency inversion** which is implemented by the **dependency injection** pattern. More on this later.

The class is now significantly different from its previous incarnation. It no longer contains the
implementation details for the whole process but instead contains the blueprint for the process.
The class models the process of transferring trade data from one format to another. This is its only
responsibility, its only concern, and the only reason that this class should change. If the process itself
changes, this class will change to reflect it. But if you decide you no longer want to retrieve data from
a file, log on to the console, or store the trades in a database, this class remains as is.

> The more observant readers may be asking where the objects injected into the TradeProcessor class come from. Well, they come from a dependency injection container. One thing that the **Single Responsibility Principle** gives rise to are lots of small classes. To assemble such small classes to work well can be a hard thing to do, and that is when dependency injection containers come to the resucue.

Since the `TradeProcessor` class now just models the workflow of converting between trade data formats, it no longer cares about where the data comes from, how it is parsed, validated and where it is stored. This means we can have different implementations of the

`DataProvider` abstraction

- Relational Database
- Text Files
- NoSql Databases
- Web services
- e.t.c

`TradeDataParser` abstraction

- CommaParser
- TabParser
- ColonParser

`TradeRepository` abstraction

- Relational Database
- Text Files
- NoSql Databases
- Web services
- e.t.c

The table below shows some of the classes implementing the above abstract classes. Notice that we can swap between any of the different implementations and `TradeProcessor` will not even know. This is what software engineers call **loose coupling**.

| Interface used by `TradeProcessor` | Public operation | Concrete implementations |
| --- | --- | --- |
| `DataProvider` | `read_trade_data(): List[str]` | `FileProvider` and `DBProvider` implement this interface. |
| `TradeDataParser` | `parse_trade_data(List[str]): List[TradeRecord]` | `CommaParser` implements this interface. |
| `TradeRepository` | `persist_trade_data(List[TradeRecord]): void` | `MongoDB` and `MySql` implement this interface. |

`TradeProcessor` still exposes `process_trades(): void` and holds the private fields
`trade_data_provider: DataProvider`, `trade_data_parser: TradeDataParser` and
`trade_data_persister: TradeRepository`. It depends on those interfaces, while the concrete classes
implement them. No additional members are specified for the concrete classes in this design.

From these relationships, we are confident that once a new storage mechanism pops up, we just roll up a class to implement the new functionality, we make sure that the class inherits from the right base class. We then inject this new class instance in `TradeProcessor`. This is the **Open Closed Principle** as we will see in the next section.

If you look so closely at these relationships, you can notice that as new requirements pop up, we get a class **big bang**. We shall solve this problem later when we look at **decorators**.

So far, we have solved 3 problems. These are:

- What happens if we need to use another data source.
- What happens if we need to store the data to a different storage.
- What happens when the business requirements call for a new parsing strategy.

**What happens if new business rules come up that need new validation rules?**

Remember that the original
`parse_trades()` method delegated responsibility for validation and for mapping. You can repeat the
process of refactoring so that the `CommaParser` class does not have more than one responsibility. At the moment, `CommaParser` is implemented as shown below

```python

class CommaParser(TradeDataParser):
    def parse_trade_data(self, trade_data : List[str]) -> List[TradeRecord]:
        trades: List[TradeRecord] = []
        for index, line in enumerate(trade_data):
            fields: List[str] = line.split(',')
            if not CommaParser.__validate_trade_data(fields, index + 1):
                continue
            trade = CommaParser.__create_trade_record(fields)
            trades.append(trade)
        return trades

    @staticmethod
    def __log_message(message: str) -> None:
        print(message)

    def __create_trade_record(self,fields: List[str]) -> TradeRecord:
        in_curr = slice(0, 3);
        out_curr = slice(3, None)
        source_curr_code = fields[0][in_curr]
        dest_curr_code = fields[0][out_curr]
        trade_amount = int(fields[1])
        trade_price = float(fields[2])

        trade_record = TradeRecord(source_curr_code, dest_curr_code,
                                   trade_amount, trade_price)
        return trade_record

    def __validate_trade_data(self, fields: List[str], index: int) -> bool:
        if len(fields) != 3:
            CommaParser.__log_message(f'Line {index} malformed. Only {len(fields)} field(s) found.')
            return False
        if len(fields[0]) != 6:
            CommaParser.__log_message(f'Trade currencies on line {index} malformed: {fields[0]}')
            return False
        try:
            trade_amount = float(fields[1])
        except ValueError:
            CommaParser.__log_message(f"Trade amount on line {index} not a valid integer: '{fields[1]}'")
            return False
        try:
            trade_price = float(fields[2])
        except ValueError:
            CommaParser.__log_message(f'Trade price on line {index} not a valid float:{fields[2]}')
            return False
        return True
```

We can see that the current implementation of `CommaParser` is not ideal. The class is having more than one responsibility to change. So we can refactor out the two methods `__validate_trade_data()` and `__create_trade_record()` into new classes since they both change for different reasons.

We will create 2 new abstractions -- `TradeMapper` (responsible for mapping validated fields into `TradeRecord` instances) and `TradeValidator` (responsible for validating the input data before creating `TradeRecord` instances).

Our new design separates parsing, validation and mapping as follows.

| Interface | Private collaborators | Public operation |
| --- | --- | --- |
| `TradeDataParser` | `trade_mapper: TradeMapper`, `trade_validator: TradeValidator` | `parse_trade_data(List[str]): List[TradeRecord]` |
| `TradeValidator` | None shown | `validate_trade_data(List[str], int): Boolean` |
| `TradeMapper` | None shown | `create_trade_record(List[str]): TradeRecord` |

The parser holds references to a validator and a mapper; these are delegation relationships, not
inheritance relationships. The validator receives the fields and line number and returns whether the
row is valid. The mapper receives validated fields and returns one `TradeRecord`.

This is a flexible design in such a way that if the parsing rules change, i.e. text is separated by tab and not ',', we just implement `TradeDataParser` in a new class.Incase the data validation rules change too, we just roll up a new class inheriting from `TradeValidator`.

Below are the implementations of the new abstractions. Note that the interface for `TradeDataParser` has changed and now takes in instances of `TradeValidator` and `TradeMapper` to help it accomplish it's task

```python

class TradeMapper(ABC):
    @abstractmethod
    def create_trade_record(self, fields: List[str]) -> TradeRecord:
        pass


class TradeValidator(ABC):
    @abstractmethod
    def validate_trade_data(self, fields: List[str], index: int) -> bool:
        pass


class TradeDataParser(ABC):
    @abstractmethod
    def parse_trade_data(self, trade_data: List[str]) -> TradeRecord:
        pass
```

And then here is the new implementation of the CommaParser in terms of these new abstractions.

```python
class CommaParser(TradeDataParser):
    def __init__(self, mapper: TradeMapper, validator: TradeValidator):
        self._mapper = mapper
        self._validator = validator

    def parse_trade_data(self, trade_data: List[str]) -> List[TradeRecord]:
        trades: List[TradeRecord] = []
        for index, line in enumerate(trade_data):
            fields: List[str] = line.split(',')
            if not self._validator.validate_trade_data(fields, index + 1):
                continue
            trade = self._mapper.create_trade_record(fields)
            trades.append(trade)
        return trades
```

Pay attention to the assignments to `self._mapper` and `self._validator` and the calls delegated to them. In the constructor, two dependencies are injected in `mapper` and `validator` and these two are used by CommaParser to parse the input assuming the string components are separated by commas hence the `split(',')`. Other parsers would implement it differently.

Below are possible implementations of the `TradeMapper` and `TradeValidator` abstractions.

```python
class SimpleTradeMapper(TradeMapper):
    def create_trade_record(self, fields: List[str]) -> TradeRecord:
        in_curr = slice(0, 3);
        out_curr = slice(3, None)
        source_curr_code = fields[0][in_curr]
        dest_curr_code = fields[0][out_curr]
        trade_amount = int(fields[1])
        trade_price = float(fields[2])

        trade_record = TradeRecord(source_curr_code, dest_curr_code,
                                   trade_amount, trade_price)
        return trade_record
```

```python

class SimpleValidator(TradeValidator):
    @staticmethod
    def __log_message(message: str) -> None:
        print(message)

    def validate_trade_data(self, fields: List[str], index: int) -> bool:
        if len(fields) != 3:
            SimpleValidator.__log_message(f'Line {index} malformed. Only {len(fields)} field(s) found.')
            return False
        if len(fields[0]) != 6:
            SimpleValidator.__log_message(f'Trade currencies on line {index} malformed: {fields[0]}')
            return False
        try:
            trade_amount = float(fields[1])
        except ValueError:
            SimpleValidator.__log_message(f"Trade amount on line {index} not a valid integer: '{fields[1]}'")
            return False
        try:
            trade_price = float(fields[2])
        except ValueError:
            SimpleValidator.__log_message(f'Trade price on line {index} not a valid float:{fields[2]}')
            return False
        return True

```

We are almost there but still we are having a smell in our design. We would love to be able to log to different destinations -- console, text file or even a database. But if you look closely at the implementations of `TradeRepository` and `TradeValidator`, the logger is hard coded and it always logs to the console.

We have to solve this problem before we run out of business. We are going to refactor this function into its abstraction. The following snippet reveals the snippet for this change.

```python
from abc import ABC, abstractmethod

class TradeLogger(ABC):
    @abstractmethod
    def log_message(self, message):
        pass


class SimpleValidator(TradeValidator):
    def __init__(self, logger: TradeLogger)->None:
        if instance(logger, TradeLogger):
            self._logger = logger
        else:
            raise AssertionError('Bad Argument')

    def validate_trade_data(self, fields: List[str], index: int) -> bool:
        if len(fields) != 3:
            self._logger.log_message(f'Line {index} malformed. Only {len(fields)} field(s) found.')
            return False
        if len(fields[0]) != 6:
            self._logger.log_message(f'Trade currencies on line {index} malformed: {fields[0]}')
            return False
        try:
            trade_amount = float(fields[1])
        except ValueError:
            self._logger.log_message(f"Trade amount on line {index} not a valid integer: '{fields[1]}'")
            return False
        try:
            trade_price = float(fields[2])
        except ValueError:
            self._logger.log_message(f'Trade price on line {index} not a valid float:{fields[2]}')
            return False
        return True
```

After all these refactorings, we finally have a collection of abstractions that work together to solve the simple problem we posed at the beginning of this chapter.

The table below shows the complete design of the abstractions and their dependencies.

| Class or interface | Private collaborators | Public operation |
| --- | --- | --- |
| `TradeProcessor` | `trade_data_parser: TradeDataParser`, `trade_data_persister: TradeRepository`, `trade_data_provider: DataProvider` | `process_trades(): void` |
| `DataProvider` | None shown | `read_trade_data(): List[str]` |
| `TradeDataParser` | `trade_mapper: TradeMapper`, `trade_validator: TradeValidator` | `parse_trade_data(List[str]): List[TradeRecord]` |
| `TradeRepository` | `trade_logger: TradeLogger` | `persist_trade_data(List[TradeRecord]): void` |
| `TradeValidator` | `trade_logger: TradeLogger` | `validate_trade_data(List[str], int): Boolean` |
| `TradeMapper` | None shown | `create_trade_record(List[str]): TradeRecord` |
| `TradeLogger` | None shown | `log_message(String): void` |

`TradeProcessor` delegates reading to `DataProvider`, parsing to `TradeDataParser`, and storage to
`TradeRepository`. The parser delegates validation to `TradeValidator` and record creation to
`TradeMapper`. Both the validator and the repository delegate logging to `TradeLogger`. Each collaborator
is accessed through its interface, so its implementation can be replaced independently.

> **Note** that none of these are concrete classes and so they can not be instantiated. To use the `TradeProcessor` class, you will need concrete implementations of all these abstractions and then you will have to wire them together to accomplish a task. **Dependency Injection** containers do this wiring.

From a monolith, we have created a miniature framework for converting trade data between formats. Congratulations!!!!!.

