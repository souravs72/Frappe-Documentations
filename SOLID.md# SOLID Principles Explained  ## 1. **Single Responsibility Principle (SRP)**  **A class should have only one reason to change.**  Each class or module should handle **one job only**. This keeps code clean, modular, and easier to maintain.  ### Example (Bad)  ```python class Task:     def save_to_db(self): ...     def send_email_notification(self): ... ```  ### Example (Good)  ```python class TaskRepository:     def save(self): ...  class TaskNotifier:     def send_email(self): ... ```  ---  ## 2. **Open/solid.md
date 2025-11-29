# SOLID Principles Explained

## 1. **Single Responsibility Principle (SRP)**

**A class should have only one reason to change.**

Each class or module should handle **one job only**. This keeps code clean, modular, and easier to maintain.

### Example (Bad)

```python
class Task:
    def save_to_db(self): ...
    def send_email_notification(self): ...
```

### Example (Good)

```python
class TaskRepository:
    def save(self): ...

class TaskNotifier:
    def send_email(self): ...
```

---

## 2. **Open/Closed Principle (OCP)**

**Software entities should be open for extension but closed for modification.**

You should be able to **add new behavior** without rewriting existing code.

### Example

Define new notifier classes instead of modifying the main logic:

```python
class EmailNotifier(NotificationRule): ...
class SMSNotifier(NotificationRule): ...
class PushNotifier(NotificationRule): ...
```

---

## 3. **Liskov Substitution Principle (LSP)**

**Subclasses should be replaceable with their parent classes without breaking functionality.**

If `B` inherits from `A`, then you should be able to use `B` anywhere `A` is expected.

---

## 4. **Interface Segregation Principle (ISP)**

**Clients should not be forced to depend on methods they do not use.**

Avoid large, bloated interfaces.

### Bad Example

```ts
interface Worker {
   work();
   eat();
   sleep();
}
```

A robot shouldn't be forced to implement `eat()` or `sleep()`.

### Good Example

```ts
interface Workable { work(); }
interface Feedable { eat(); }
interface Restable { sleep(); }
```

---

## 5. **Dependency Inversion Principle (DIP)**

**Depend on abstractions, not concrete implementations.**

High-level modules should not be tightly coupled to low-level modules.

### Bad Example

```python
class TaskService:
    self.repo = MySQLRepository()   # Hard dependency
```

### Good Example

```python
class TaskService:
    def __init__(self, repository: ITaskRepository):
        self.repo = repository
```

Now you can easily swap in-memory, MySQL, or Postgres repositories.

---

## 📌 Summary

SOLID principles help your code become:

* More maintainable
* Easier to extend
* Easier to test
* Less tightly coupled
* Cleaner and more predictable
