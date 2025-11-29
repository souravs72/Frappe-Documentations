## Testing Guide

This teaching-oriented guide walks through Frappe test development for `CG User`, `CG Task Definition`, and `CG Task Instance` in increasing levels of depth and complexity.

---

# Table of Contents

1. [Introduction to Frappe Testing](#1-introduction-to-frappe-testing)
2. [Beginner: Core Concepts & Simple Test Patterns](#2-beginner-core-concepts--simple-test-patterns)
   - 2.1 [What is a Test?](#21-what-is-a-test)
   - 2.2 [How Frappe Tests Run](#22-how-frappe-tests-run)
   - 2.3 [Writing Your First Test](#23-writing-your-first-test)
3. [Intermediate: Shared Test Helpers & Test Data Creation](#3-intermediate-shared-test-helpers--test-data-creation)
   - 3.1 [Using Helper Functions](#31-using-helper-functions)
   - 3.2 [Test Setup and Teardown](#32-test-setup-and-teardown)
   - 3.3 [Testing Validations and Permissions](#33-testing-validations-and-permissions)
4. [Advanced: Test Patterns, Mocking, and Integrated Workflows](#4-advanced-test-patterns-mocking-and-integrated-workflows)
   - 4.1 [Unit vs Integration Tests](#41-unit-vs-integration-tests)
   - 4.2 [Mocking External Dependencies](#42-mocking-external-dependencies)
   - 4.3 [Performance & Side-Effects](#43-performance--side-effects)
   - 4.4 [End-to-End Test Scenarios](#44-end-to-end-test-scenarios)
5. [Expert: Best Practices, Debugging, and Test Architecture](#5-expert-best-practices-debugging-and-test-architecture)
   - 5.1 [Test Design Principles](#51-test-design-principles)
   - 5.2 [Debugging and Flaky Test Fixes](#52-debugging-and-flaky-test-fixes)
   - 5.3 [Teaching Tips for Different Levels](#53-teaching-tips-for-different-levels)
   - 5.4 [Decision Trees for Test Types](#54-decision-trees-for-test-types)
6. [Quick Reference & Cheat Sheet](#6-quick-reference--cheat-sheet)

---

## 1. Introduction to Frappe Testing

**What is Frappe Testing?**  
Testing in Frappe means writing reusable scripts that check your code works as intended—validations, workflow, and permissions are handled properly in your DocTypes and business logic.

**Why Test?**  
- Prevent bugs before they reach users.
- Enable safe upgrades and refactoring.
- Document business rules for future maintainers.

**Testing Levels:**
- **Unit test:** Isolated check of a function/class logic.
- **Integration test:** Real DB, real DocTypes, multiple components working together.
- **Workflow/end-to-end test:** Simulates actual business user interactions.

---

## 2. Beginner: Core Concepts & Simple Test Patterns

### 2.1 What is a Test?

A **test** is a function or script that checks for correct or incorrect behavior, e.g., a validation works, a document saves, or an error is triggered if you break a rule.

**Typical test structure:**
- Prepare test data ("fixtures")
- Run a scenario (call a function, save a document)
- Assert expected behavior (checks: `self.assertEqual(...)`, `with self.assertRaises(...)`)

### 2.2 How Frappe Tests Run

- Written as Python classes/functions (usually inside `test_*.py` files).
- Use `FrappeTestCase`/`UnitTestCase` as base class.
- Run tests using the Frappe CLI:  
  `bench --site <yoursite> run-tests --module <full.module.path>`

### 2.3 Writing Your First Test

Start simple! Example: Test user creation.

```python
from frappe.tests.utils import FrappeTestCase
from clapgrow_app.tests.utils import create_cg_user

class TestSimpleUser(FrappeTestCase):
    def test_create_user(self):
        user = create_cg_user(email="beginner@example.com")
        self.assertEqual(user.email, "beginner@example.com")
```

**Try it out:**  
- Save to `clapgrow_app/clapgrow_app/doctype/cg_user/test_cg_user.py`
- Run `bench ...` as above.

---

## 3. Intermediate: Shared Test Helpers & Test Data Creation

### 3.1 Using Helper Functions

Helpers in `clapgrow_app/tests/utils.py` keep your tests clean and DRY.  

**Examples:**  
- `create_cg_company()`
- `create_cg_department()`
- `create_cg_user()`
- `create_task_definition()`
- `create_task_instance()`

**Benefits:**  
- One-liner to generate complex test data.
- Automatically handles permissions, relationships, default values.

### 3.2 Test Setup and Teardown

Good tests isolate each scenario. Use these patterns:

```python
class TestCGTaskDefinition(FrappeTestCase):
    def setUp(self):
        self.company = create_cg_company("My Test Co")
        self.user = create_cg_user(email="myuser@test.co", company_id=self.company.name)

    def tearDown(self):
        frappe.db.rollback()
```

### 3.3 Testing Validations and Permissions

**Validation test example:**

```python
def test_requires_due_date(self):
    with self.assertRaises(frappe.ValidationError):
        task_def = create_task_definition(
            task_name="Test",
            assigned_to=self.user.name,
            frequency="Daily",
            do_not_save=True
        )
        task_def.save()  # Should raise error
```

**Permission test example:**

```python
def test_member_cannot_assign_to_admin(self):
    frappe.set_user(self.user.email)
    # Assume role setup disables this
    with self.assertRaises(frappe.ValidationError):
        create_task_definition(
            task_name="Test",
            assigned_to="admin@test.co",
            assignee=self.user.email,
            frequency="Daily",
            due_date=self.get_future_due_date(),
        )
```

---

## 4. Advanced: Test Patterns, Mocking, and Integrated Workflows

### 4.1 Unit vs Integration Tests

**Unit Test:** Fast, checks pure functions.  
**Integration Test:** Real documents, database, permissions.

**Examples in context:**  
- Test a calculation function (unit)
- Test DocType validation (integration)
- Both needed for robust apps!

### 4.2 Mocking External Dependencies

**Why mock?**  
- Avoid sending real emails, HTTP calls.
- Speed and cost.

**Mock example:** (prevent real WhatsApp messages)

```python
from unittest.mock import patch

def test_sends_whatsapp_message(self):
    with patch('clapgrow_app.api.whatsapp.requests.post') as mock_post:
        mock_post.return_value.status_code = 200
        # ... call function, check mock_post.assert_called_once()
```

**DO NOT mock DocType logic, DB, permissions—use real helpers instead!**

### 4.3 Performance & Side-Effects

- Use `do_not_save=True` for in-memory tests.
- Use DB rollback after each test (`tearDown`) to avoid pollution.
- Don’t run slow jobs; mock job queueing instead.

### 4.4 End-to-End Test Scenarios

Combine everything:  
- Setup org structure and users.
- Create a recurring task.
- Simulate scheduler, pause/resume flows.
- Assert DB state before and after.

Example: [test_scheduler_updates_status_priority_map](#test_scheduler_updates_status_priority_map)

---

## 5. Expert: Best Practices, Debugging, and Test Architecture

### 5.1 Test Design Principles

- **Use helpers** for fixtures, avoid manual document wiring.
- **Test one thing per test method** (“single responsibility”).
- **Roll back DB between tests** for isolation.
- **Mock only where external APIs/side-effects exist**.
- **Write both unit and integration tests** for key business logic.

### 5.2 Debugging and Flaky Test Fixes

Common problems:
- Permissions: Use ignore_permissions and correct user setup.
- Dates: Always use relative future/past (add_to_date) instead of fixed ones.
- DocType existence: Use helpers to guarantee test data.
- Flaky tests: Narrow scope, avoid dependencies on previous DB state.

### 5.3 Teaching Tips for Different Levels

**Beginner:**  
- Explain every helper and assertion.
- Use plenty of “run this and see the result” tasks.
- Visualize what each test is intended to prove.

**Intermediate:**  
- Use small group exercises to expand tests (add validation, permission checks).
- Compare and contrast unit vs integration tests.
- Encourage reading docs and related CI output for test runs.

**Advanced:**  
- Assign refactoring tasks (extract helpers, modularize tests).
- Introduce mocks for complex external APIs.
- Discuss architectural decisions: when to test what, and why.

**Expert:**  
- Pair-program creation of large, reusable test suites.
- Explore test failures and debugging workflows.
- Encourage documentation through tests.

### 5.4 Decision Trees for Test Types

- **External API call?** → Use mock.
- **Pure function, no DB?** → Unit test.
- **Business logic, affects DocType or DB?** → Integration test.
- **Permission, workflow, validation?** → Integration test, helpers, real users/roles.

---

## 6. Quick Reference & Cheat Sheet

### Key Helpers

| Helper                     | Purpose                                      |
|----------------------------|----------------------------------------------|
| create_cg_company          | Create/lookup a company                      |
| create_cg_department       | Create/lookup a department (requires company)|
| create_cg_branch           | Create/lookup a branch with work hours       |
| create_gender              | Pre-load standard genders, needed for users  |
| create_cg_user             | Create/lookup user by email                  |
| create_task_instance       | Create onetime task instance (assign, due, etc.)|
| create_recurrence_type     | Build recurrence for a task definition       |
| create_task_definition     | Create recurring tasks, generates instances  |

### Handy Test Patterns

```python
with self.assertRaises(frappe.ValidationError):
    # code that should fail
    doc.save()

self.assertEqual(doc.field, expected_value)

frappe.set_user(user.email)    # Change current user
frappe.db.rollback()           # Cleanup after test
```

### Common Troubleshooting

| Symptom                  | Probable Cause/Action              |
|--------------------------|------------------------------------|
| PermissionError          | Fix user setup, use ignore_permissions|
| Document missing         | Use helper to create before test   |
| Date errors              | Use future/past calculation helpers|
| Flaky test               | Isolate data, always rollback     |

---

## Teaching Module Summary

When teaching:
- Start with **helper-guided simple tests**; visually walk through code and assertions.
- Connect **fixtures** to real-world business context (company, user, task).
- Progress to **validations/permission tests**; discuss “why” these errors occur.
- Introduce **units, integrations, mocks** level by level.
- End with **troubleshooting and architecture**: Why well-structured, DRY tests matter for large apps.

With this guide, your students can confidently explore Frappe testing, stepwise from absolute beginner to seasoned backend engineer!

---
