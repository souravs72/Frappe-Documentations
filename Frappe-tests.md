## CG User & Task Testing Guide

This document explains how `CG User`, `CG Task Definition`, and `CG Task Instance` tests are structured using the shared helpers in `clapgrow_app/tests/utils.py`.

- **Target audience**: Anyone adding/modifying backend tests around CG users and tasks.
- **Key idea**: Centralise all "create company/branch/user/task" boilerplate in `tests/utils.py` so your tests stay short and focused on business rules.

---

## 1. Shared test helpers (`tests/utils.py`) - Complete Examples

Helpers live in `clapgrow_app/tests/utils.py`. Below are detailed examples for each helper function.

### 1.1 `_save_doc(doc, do_not_save=False, ignore_permissions=None)`

**What it does**: Internal wrapper that handles saving documents in tests with proper permission handling.

**Example 1: Normal save (automatic permission handling)**

```python
# Inside a helper function - you rarely call this directly
doc = frappe.new_doc("CG Company")
doc.company_name = "My Company"
_save_doc(doc)  # Automatically uses ignore_permissions=True in test mode
```

**Example 2: Skip save for validation tests**

```python
# When testing that save() raises an error
doc = create_task_definition(
    task_name="Test",
    assigned_to="user@example.com",
    do_not_save=True  # Document created but not saved
)
# Now you can manually call save() and expect it to raise
with self.assertRaises(frappe.ValidationError):
    doc.save()
```

**Example 3: Force permission check**

```python
# Rare: when you want to test permission errors
doc = create_cg_user(email="test@example.com", ignore_permissions=False)
# This will raise if current user lacks permission
```

---

### 1.2 `create_cg_company(company_name=None, **kwargs)`

**What it does**: Creates or reuses a `CG Company`. Idempotent - safe to call multiple times.

**Example 1: Basic usage (uses default)**

```python
company = create_cg_company()
# Creates: "Clapgrow Technology Private Ltd."
# Returns: CG Company document
print(company.name)  # "Clapgrow Technology Private Ltd."
```

**Example 2: Custom company name**

```python
company = create_cg_company(company_name="Acme Corp")
# Creates: "Acme Corp"
# If called again with same name, returns existing company
company2 = create_cg_company(company_name="Acme Corp")
assert company.name == company2.name  # Same document
```

**Example 3: In test setup (typical pattern)**

```python
def setUp(self):
    self.company = create_cg_company("My Test Company")
    self.company_id = self.company.name
    # Now use self.company_id in other helpers
```

**Example 4: Without saving (for validation tests)**

```python
company = create_cg_company(
    company_name="Test Co",
    do_not_save=True
)
# Document exists in memory but not in DB
```

---

### 1.3 `create_gender()`

**What it does**: Seeds the `Gender` doctype with standard values.

**Example: Call once in setUp**

```python
def setUp(self):
    create_gender()  # Creates Male, Female, Others if not exist
    # Now your user creation won't fail on gender validation
    user = create_cg_user(email="test@example.com", gender="Female")
```

**Note**: This is idempotent - safe to call multiple times. Usually called once per test class.

---

### 1.4 `create_cg_department(department_name=None, company_id=None, **kwargs)`

**What it does**: Creates or reuses a `CG Department` linked to a company.

**Example 1: Basic usage (uses defaults)**

```python
dept = create_cg_department()
# Creates: "Test Department" linked to default company
# Returns: CG Department document
```

**Example 2: Custom department with specific company**

```python
company = create_cg_company("Acme Corp")
dept = create_cg_department(
    department_name="Engineering",
    company_id=company.name
)
# Creates: "Engineering" linked to "Acme Corp"
```

**Example 3: In test setup (typical pattern)**

```python
def setUp(self):
    self.company = create_cg_company("Test Company")
    self.department = create_cg_department(
        department_name="Test Department",
        company_id=self.company.name
    )
    # Now use self.department.name when creating users
```

**Example 4: Reusing existing department**

```python
# First call
dept1 = create_cg_department(
    department_name="Sales",
    company_id="Acme Corp"
)

# Second call with same params - returns existing
dept2 = create_cg_department(
    department_name="Sales",
    company_id="Acme Corp"
)
assert dept1.name == dept2.name  # Same document
```

---

### 1.5 `create_cg_branch(branch_name=None, company_id=None, start_time=None, end_time=None, **kwargs)`

**What it does**: Creates or reuses a `CG Branch` with working hours.

**Example 1: Basic usage (uses defaults)**

```python
branch = create_cg_branch()
# Creates: "Test Branch" with 09:00:00 - 18:00:00
```

**Example 2: Custom branch with specific hours**

```python
branch = create_cg_branch(
    branch_name="Night Shift",
    company_id="Acme Corp",
    start_time="22:00:00",
    end_time="06:00:00"
)
```

**Example 3: In test setup (typical pattern)**

```python
def setUp(self):
    self.company = create_cg_company("Test Company")
    self.branch = create_cg_branch(
        branch_name="Test Branch",
        company_id=self.company.name,
        start_time="09:00:00",
        end_time="18:00:00"
    )
```

---

### 1.6 `create_cg_user(first_name=None, email=None, phone=None, **kwargs)` ⭐ MOST USED

**What it does**: Creates or reuses a `CG User` identified by email. This is the most frequently used helper.

**Example 1: Basic usage (minimal params)**

```python
user = create_cg_user()
# Creates user with:
# - email: "test@example.in"
# - phone: "+91 9999999999"
# - first_name: "CG Test User"
# - role: "ROLE-Admin"
# - enabled: 1
# - Default company/department/branch
```

**Example 2: Custom email (most common pattern)**

```python
user = create_cg_user(email="john@example.com")
# Creates new user OR returns existing if email matches
# Email is the unique identifier
```

**Example 3: Full user setup (typical in test setUp)**

```python
def setUp(self):
    # Setup org structure first
    self.company = create_cg_company("Test Company")
    self.department = create_cg_department(
        department_name="Engineering",
        company_id=self.company.name
    )
    self.branch = create_cg_branch(
        branch_name="HQ",
        company_id=self.company.name
    )

    # Create users with specific roles
    self.admin_user = create_cg_user(
        first_name="Admin",
        email="admin@test.com",
        phone="+91 9876543210",
        role="ROLE-Admin",
        company_id=self.company.name,
        department_id=self.department.name,
        branch_id=self.branch.name
    )

    self.member_user = create_cg_user(
        first_name="Member",
        email="member@test.com",
        role="ROLE-Member",  # Different role
        company_id=self.company.name,
        department_id=self.department.name,
        branch_id=self.branch.name
    )
```

**Example 4: Reusing existing user (idempotent behavior)**

```python
# First call
user1 = create_cg_user(email="test@example.com", first_name="John")

# Second call with same email - returns existing user
user2 = create_cg_user(email="test@example.com", first_name="Jane")
# Note: user2.first_name is still "John" (existing data preserved)
# Only email matters for lookup
```

**Example 5: User with custom gender**

```python
create_gender()  # Ensure genders exist
user = create_cg_user(
    email="sarah@example.com",
    gender="Female",
    first_name="Sarah"
)
```

**Example 6: User without saving (for validation tests)**

```python
user = create_cg_user(
    email="test@example.com",
    do_not_save=True
)
# User exists in memory, can test validations before save
with self.assertRaises(frappe.ValidationError):
    user.phone = "invalid"  # Example validation
    user.save()
```

---

### 1.7 `create_task_instance(task_name=None, assigned_to=None, **kwargs)`

**What it does**: Creates a one-time `CG Task Instance`. This is a thin wrapper around `create_task_document`.

**Example 1: Basic one-time task**

```python
task = create_task_instance(
    task_name="Complete report",
    assigned_to="user@example.com"
)
# Creates task with:
# - task_type: "Onetime"
# - priority: "Medium"
# - enabled: 1
```

**Example 2: Task with due date and priority**

```python
from frappe.utils import add_to_date

due_date = add_to_date(days=1)  # Tomorrow
task = create_task_instance(
    task_name="Urgent task",
    assigned_to="user@example.com",
    due_date=due_date,
    priority="Critical"
)
```

**Example 3: Task with assignee (different from assigned_to)**

```python
task = create_task_instance(
    task_name="Task",
    assigned_to="member@example.com",  # Who does the work
    assignee="admin@example.com"        # Who assigned it
)
```

**Example 4: In test class (typical pattern)**

```python
def test_task_creation(self):
    user = create_cg_user(email="test@example.com")
    task = create_task_instance(
        task_name="Test Task",
        assigned_to=user.name,
        due_date=self.get_future_due_date(),
        priority="High"
    )
    self.assertEqual(task.status, "Upcoming")
```

**Example 5: Task without saving**

```python
task = create_task_instance(
    task_name="Test",
    assigned_to="user@example.com",
    do_not_save=True
)
# Test validations before save
with self.assertRaises(frappe.ValidationError):
    task.due_date = None  # If due_date is required
    task.save()
```

---

### 1.8 `create_recurrence_type(frequency, **kwargs)`

**What it does**: Returns a dictionary for a `CG Recurrence Type` child table row. Used internally by `create_task_definition`.

**Example 1: Daily recurrence**

```python
recurrence = create_recurrence_type("Daily")
# Returns: {"doctype": "CG Recurrence Type", "frequency": "Daily"}
```

**Example 2: Weekly recurrence with specific days**

```python
recurrence = create_recurrence_type(
    frequency="Weekly",
    week_days="Monday,Wednesday,Friday"
)
# Returns dict with week_days set
```

**Example 3: Monthly recurrence**

```python
recurrence = create_recurrence_type(
    frequency="Monthly",
    month_days=15  # 15th of every month
)
```

**Example 4: Custom recurrence (every 2 days)**

```python
recurrence = create_recurrence_type(
    frequency="Custom",
    custom_type="Daily",
    interval=2,
    unit="Days"
)
```

**Example 5: Custom recurrence (every 12 hours)**

```python
recurrence = create_recurrence_type(
    frequency="Custom",
    custom_type="Daily",
    interval=12,
    unit="Hours"
)
```

**Note**: You rarely call this directly. It's used by `create_task_definition`.

---

### 1.9 `create_task_definition(task_name, assigned_to, **kwargs)` ⭐ COMPLEX HELPER

**What it does**: Creates a `CG Task Definition` (recurring task) with full configuration. This is the most complex helper.

**Example 1: Basic daily recurring task**

```python
from frappe.utils import add_to_date

user = create_cg_user(email="user@example.com")
task_def = create_task_definition(
    task_name="Daily standup",
    assigned_to=user.name,
    frequency="Daily",
    due_date=add_to_date(days=1)
)
# Creates task definition and automatically generates task instances
```

**Example 2: Weekly task (Monday, Wednesday, Friday)**

```python
task_def = create_task_definition(
    task_name="Weekly meetings",
    assigned_to=user.name,
    frequency="Weekly",
    week_days="Monday,Wednesday,Friday",
    due_date=add_to_date(days=1)
)
```

**Example 3: Monthly task (15th of every month)**

```python
task_def = create_task_definition(
    task_name="Monthly report",
    assigned_to=user.name,
    frequency="Monthly",
    month_days=15,
    due_date=add_to_date(days=1)
)
```

**Example 4: Task with checklist**

```python
checklist = [
    {"checklist_item": "Review code", "description": "Check PR"},
    {"checklist_item": "Update docs", "description": "Update README"}
]

task_def = create_task_definition(
    task_name="Code review task",
    assigned_to=user.name,
    frequency="Daily",
    due_date=add_to_date(days=1),
    checklist=checklist
)
# Generated task instances will have these checklist items
```

**Example 5: Task with subtasks**

```python
subtasks = [
    {"subtask_name": "Design", "assigned_to": user.name},
    {"subtask_name": "Implement", "assigned_to": user.name}
]

task_def = create_task_definition(
    task_name="Feature development",
    assigned_to=user.name,
    frequency="Daily",
    due_date=add_to_date(days=1),
    subtasks=subtasks
)
# Generated instances will have these subtasks
```

**Example 6: Task with holiday behavior**

```python
task_def = create_task_definition(
    task_name="Daily task",
    assigned_to=user.name,
    frequency="Daily",
    due_date=add_to_date(days=1),
    holiday_behaviour="Next Working Date"  # Skip holidays
)
```

**Example 7: Task with reminders**

```python
task_def = create_task_definition(
    task_name="Reminder task",
    assigned_to=user.name,
    frequency="Daily",
    due_date=add_to_date(days=1),
    reminder_enabled=1,
    reminder_unit="Hours",
    reminder_before_value=2,
    reminder_start_time="09:00:00",
    reminder_times=1
)
```

**Example 8: Custom frequency (every 3 days)**

```python
task_def = create_task_definition(
    task_name="Every 3 days",
    assigned_to=user.name,
    frequency="Custom",
    custom_type="Daily",
    interval=3,
    unit="Days",
    due_date=add_to_date(days=1)
)
```

**Example 9: Task without saving (for validation tests)**

```python
task_def = create_task_definition(
    task_name="Test",
    assigned_to=user.name,
    frequency="Daily",
    do_not_save=True  # No due_date - should fail validation
)
with self.assertRaises(frappe.ValidationError):
    task_def.save()  # Missing due_date
```

**Example 10: Full setup in test (typical pattern)**

```python
def test_recurring_task(self):
    user = create_cg_user(email="member@test.com")
    due_date = add_to_date(days=1)

    task_def = create_task_definition(
        task_name="Daily task",
        assigned_to=user.name,
        frequency="Daily",
        due_date=due_date,
        priority="High",
        description="Important daily task"
    )

    # Assert instances were created
    instances = frappe.get_all(
        "CG Task Instance",
        filters={"task_definition_id": task_def.name}
    )
    self.assertGreater(len(instances), 0)
```

---

## 2. CG Task Definition tests – Complete Examples

File: `clapgrow_app/clapgrow_app/doctype/cg_task_definition/test_cg_task_definition.py`

### 2.1 Test Class Structure

**Base class**: `TestCGTaskDefinition(FrappeTestCase)`

**Example: Complete setUp and tearDown pattern**

```python
class TestCGTaskDefinition(FrappeTestCase):
    def setUp(self):
        # 1. Create test data (org structure + users)
        self.setup_test_data()

        # 2. Set current user (for permission testing)
        if hasattr(self, "admin_user"):
            frappe.set_user(self.admin_user.email)

        # 3. Call parent setUp
        super().setUp()

    def tearDown(self):
        # Rollback all DB changes after each test
        frappe.db.rollback()
        super().tearDown()

    def setup_test_data(self):
        """Create consistent test fixtures used by all tests."""
        # Create company
        self.company = create_cg_company("Clapgrow Technology Private Ltd.")
        self.company_id = self.company.name

        # Create department and branch
        self.department = create_cg_department(
            department_name="Test Department",
            company_id=self.company_id,
        )
        self.branch = create_cg_branch(
            branch_name="Test Branch",
            company_id=self.company_id,
        )

        # Create users with different roles
        self.admin_user = create_cg_user(
            first_name="Admin User",
            email="admin@test.cg",
            phone="+91 9999999001",
            role="ROLE-Admin",
            company_id=self.company_id,
            department_id=self.department.name,
            branch_id=self.branch.name,
        )

        self.team_lead_user = create_cg_user(
            first_name="Team Lead",
            email="teamlead@test.cg",
            phone="+91 9999999002",
            role="ROLE-Team Lead",
            company_id=self.company_id,
            department_id=self.department.name,
            branch_id=self.branch.name,
        )

        self.member_user = create_cg_user(
            first_name="Member",
            email="member@test.cg",
            phone="+91 9999999003",
            role="ROLE-Member",
            company_id=self.company_id,
            department_id=self.department.name,
            branch_id=self.branch.name,
        )

    def get_future_due_date(self, days=1, hours=0):
        """Helper to get future dates."""
        return add_to_date(days=cint(days), hours=flt(hours))

    def get_past_due_date(self, days=1, hours=0):
        """Helper to get past dates."""
        return add_to_date(days=-cint(days), hours=-flt(hours))

    def count_task_instances(self, task_definition_id):
        """Helper to count generated instances."""
        return frappe.db.count("CG Task Instance", {
            "task_definition_id": task_definition_id
        })
```

---

### 2.2 Validation Test Examples

**Example 1: Test that recurring task requires due_date**

```python
def test_recurring_task_requires_due_date(self):
    """Test that recurring tasks require a due date."""
    with self.assertRaises(frappe.ValidationError):
        task_def = create_task_definition(
            task_name="Test Task",
            assigned_to=self.member_user.name,
            frequency="Daily",
            do_not_save=True,  # Don't save yet
        )
        task_def.save()  # This should raise ValidationError
```

**Example 2: Test that recurring task requires recurrence_type**

```python
def test_recurring_task_requires_recurrence_type(self):
    """Test that recurring tasks require a recurrence type."""
    with self.assertRaises(frappe.ValidationError):
        task_def = create_task_definition(
            task_name="Test Task",
            assigned_to=self.member_user.name,
            due_date=self.get_future_due_date(),
            # Missing frequency - should fail
            do_not_save=True,
        )
        task_def.save()
```

**Example 3: Test pause validation**

```python
def test_pause_requires_start_date(self):
    """Test that pausing requires a start date."""
    task_def = create_task_definition(
        task_name="Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=self.get_future_due_date(),
    )

    task_def.is_paused = 1
    # Missing pause_start_date - should fail
    with self.assertRaises(frappe.ValidationError):
        task_def.save()
```

**Example 4: Test pause date validation**

```python
def test_pause_end_date_must_be_after_start_date(self):
    """Test that pause end date must be after start date."""
    task_def = create_task_definition(
        task_name="Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=self.get_future_due_date(),
    )

    task_def.is_paused = 1
    task_def.pause_start_date = self.get_future_due_date(2)  # Day 2
    task_def.pause_end_date = self.get_future_due_date(1)    # Day 1 - invalid!

    with self.assertRaises(frappe.ValidationError):
        task_def.save()
```

---

### 2.3 Permission Test Examples

**Example 1: Test member can only assign to self**

```python
def test_member_can_only_assign_to_self(self):
    """Test that members can only assign tasks to themselves by default."""
    # Disable permissions for member role
    frappe.db.set_value("CG Role", self.member_user.role, "assign_team_lead", 0)
    frappe.db.set_value("CG Role", self.member_user.role, "assign_admin", 0)

    with self.assertRaises(frappe.ValidationError):
        create_task_definition(
            task_name="Test Task",
            assigned_to=self.team_lead_user.name,  # Trying to assign to team lead
            assignee=self.member_user.email,       # But member is assigning
            frequency="Daily",
            due_date=self.get_future_due_date(),
        )
```

**Example 2: Test member can assign to admin when role allows**

```python
def test_member_can_only_assign_to_admin_when_role_allows(self):
    """Test that members can assign to admin when role permission is set."""
    # Enable permission while still admin
    frappe.db.set_value("CG Role", self.member_user.role, "assign_admin", 1)

    # Switch to member user
    frappe.set_user(self.member_user.email)

    # Now should work
    task_def = create_task_definition(
        task_name="Test Task",
        assigned_to=self.admin_user.email,
        assignee=self.member_user.email,
        frequency="Daily",
        due_date=self.get_future_due_date(),
    )

    self.assertIsNotNone(task_def.name)
    self.assertGreater(self.count_task_instances(task_def.name), 0)
```

**Example 3: Test team lead can assign within department**

```python
def test_team_lead_can_assign_within_department(self):
    """Test that team leads can assign tasks within their department."""
    frappe.set_user(self.team_lead_user.name)

    task_def = create_task_definition(
        task_name="Test Task",
        assigned_to=self.member_user.name,  # Member in same department
        frequency="Daily",
        due_date=self.get_future_due_date(),
    )

    self.assertIsNotNone(task_def.name)
```

---

### 2.4 Instance Generation Test Examples

**Example 1: Test daily instance generation**

```python
def test_daily_instance_generation(self):
    """Test that daily tasks generate instances correctly."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Daily Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=due_date,
    )

    # Check that instances were created
    instance_count = self.count_task_instances(task_def.name)
    self.assertGreater(instance_count, 0, "Should create at least one daily instance")

    # Verify generated_till_date is set
    self.assertIsNotNone(task_def.generated_till_date)
```

**Example 2: Test weekly instance generation**

```python
def test_weekly_instance_generation(self):
    """Test that weekly tasks generate instances for specified weekdays."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Weekly Test Task",
        assigned_to=self.member_user.name,
        frequency="Weekly",
        due_date=due_date,
        week_days="Monday,Wednesday,Friday",
    )

    instance_count = self.count_task_instances(task_def.name)
    self.assertGreaterEqual(instance_count, 0)

    # Verify instances are on correct weekdays
    if instance_count > 0:
        instances = frappe.get_all(
            "CG Task Instance",
            filters={"task_definition_id": task_def.name},
            fields=["due_date", "name"],
        )

        for instance in instances:
            due_dt = get_datetime(instance.due_date)
            weekday_name = WEEKDAYS[due_dt.weekday()]
            self.assertIn(weekday_name, ["Monday", "Wednesday", "Friday"])
```

**Example 3: Test instances include checklist**

```python
def test_instances_include_checklist(self):
    """Test that generated instances include checklist items."""
    due_date = self.get_future_due_date()
    checklist = [
        {"checklist_item": "Item 1", "description": "First item"},
        {"checklist_item": "Item 2", "description": "Second item"},
    ]

    task_def = create_task_definition(
        task_name="Checklist Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=due_date,
        checklist=checklist,
    )

    # Get first instance
    instances = frappe.get_all(
        "CG Task Instance",
        filters={"task_definition_id": task_def.name},
        fields=["name"],
        limit=1,
    )

    if instances:
        instance = frappe.get_doc("CG Task Instance", instances[0].name)
        self.assertEqual(len(instance.checklist), 2)
        self.assertEqual(instance.checklist[0].checklist_item, "Item 1")
```

**Example 4: Test instances include subtasks**

```python
def test_instances_include_subtasks(self):
    """Test that generated instances include subtasks."""
    due_date = self.get_future_due_date()
    subtasks = [
        {"subtask_name": "Subtask 1", "assigned_to": self.member_user.name},
        {"subtask_name": "Subtask 2", "assigned_to": self.member_user.name},
    ]

    task_def = create_task_definition(
        task_name="Subtask Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=due_date,
        subtasks=subtasks,
    )

    # Get first instance
    instances = frappe.get_all(
        "CG Task Instance",
        filters={"task_definition_id": task_def.name},
        fields=["name"],
        limit=1,
    )

    if instances:
        instance = frappe.get_doc("CG Task Instance", instances[0].name)
        self.assertEqual(instance.has_subtask, 1)
        self.assertEqual(len(instance.subtask), 2)
```

---

### 2.5 Pause/Resume Test Examples

**Example 1: Test pausing a task**

```python
def test_pause_task(self):
    """Test pausing a recurring task."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Pause Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=due_date,
    )

    # Pause the task
    task_def.is_paused = 1
    task_def.pause_start_date = add_to_date(days=1)
    task_def.pause_end_date = add_to_date(days=3)
    task_def.save()

    task_def.reload()
    self.assertEqual(task_def.is_paused, 1)
    self.assertIsNotNone(task_def.pause_start_date)
```

**Example 2: Test resuming a paused task**

```python
def test_resume_task(self):
    """Test resuming a paused recurring task."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Resume Test Task",
        assigned_to=self.member_user.email,
        frequency="Daily",
        due_date=due_date,
    )

    # Pause first as admin_user
    frappe.set_user(self.admin_user.email)
    task_def.is_paused = 1
    task_def.pause_start_date = add_to_date(days=1)
    task_def.pause_end_date = add_to_date(days=2)
    task_def.save()

    # Resume as the assigned user (member_user)
    frappe.set_user(self.member_user.email)
    task_def.resume_recurring_task(generate_missed_instances=False)

    task_def.reload()
    self.assertEqual(task_def.is_paused, 0)
    self.assertIsNone(task_def.pause_start_date)
    self.assertIsNone(task_def.pause_end_date)
```

**Example 3: Test pause deletes future instances**

```python
def test_pause_deletes_future_instances(self):
    """Test that pausing deletes future incomplete instances."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Pause Delete Test Task",
        assigned_to=self.member_user.name,
        assignee=self.admin_user.email,
        frequency="Daily",
        due_date=due_date,
    )

    # Use the assignee from the task definition
    frappe.set_user(task_def.assignee)
    pause_start = add_to_date(days=1)
    pause_end = add_to_date(pause_start, days=2)

    initial_instances = self.count_task_instances(task_def.name)
    self.assertGreater(initial_instances, 0)

    # Pause the task
    task_def.is_paused = 1
    task_def.pause_start_date = pause_start
    task_def.pause_end_date = pause_end
    task_def.save()

    # Verify instances in pause window are deleted
    instances_in_pause_window = frappe.get_all(
        "CG Task Instance",
        filters={
            "task_definition_id": task_def.name,
            "due_date": ["between", [pause_start, pause_end]]
        },
        fields=["name"],
    )
    self.assertEqual(
        len(instances_in_pause_window),
        0,
        "Paused task should not retain instances within the pause range.",
    )
```

---

### 2.6 Holiday Handling Test Examples

**Example 1: Test ignore holiday behavior**

```python
def test_ignore_holiday_behaviour(self):
    """Test that tasks with 'Ignore Holiday' create instances on holidays."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Ignore Holiday Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=due_date,
        holiday_behaviour="Ignore Holiday",
    )

    instance_count = self.count_task_instances(task_def.name)
    self.assertGreaterEqual(instance_count, 0)
```

**Example 2: Test next working date behavior**

```python
def test_next_working_date_behaviour(self):
    """Test that tasks with 'Next Working Date' adjust for holidays."""
    # Update branch with default week off holiday list
    branch = self.member_user.branch_id
    branch_doc = frappe.get_doc("CG Branch", branch)
    branch_doc.auto_generate_weekly_offs = 1
    branch_doc.default_weekly_off_days = "Sat,Sun"

    current_user = frappe.session.user
    try:
        frappe.set_user("Administrator")
        branch_doc.save(ignore_permissions=True)
    finally:
        frappe.set_user(current_user)
    branch_doc.reload()

    task_def = create_task_definition(
        task_name="Daily Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=get_datetime(),
        holiday_behaviour="Next Working Date",
    )

    # Validate task instance dates skip holidays
    task_ins = frappe.get_all(
        "CG Task Instance",
        filters={"task_definition_id": task_def.name},
        fields=["name", "due_date"]
    )

    week_range = get_week_range(get_datetime(), mode="current_week")
    holiday_dates = task_def.get_holiday_dates(
        week_range["start"],
        week_range["end"],
        use_cache=True,
    )

    # Validate task instance date
    for task in task_ins:
        task_ins_due_date = get_datetime(task.due_date).date()
        self.assertNotIn(
            task_ins_due_date,
            holiday_dates,
            "Should not create task instance in holidays"
        )
```

---

### 2.7 Deletion Test Examples

**Example 1: Test deletion deletes incomplete instances**

```python
def test_deletion_deletes_incomplete_instances(self):
    """Test that deleting task definition deletes incomplete instances."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Delete Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=due_date,
    )

    initial_count = self.count_task_instances(task_def.name)
    self.assertGreater(initial_count, 0)

    # Delete the task definition
    frappe.set_user(self.admin_user.email)
    frappe.delete_doc("CG Task Definition", task_def.name)

    # Verify instances are deleted
    final_count = self.count_task_instances(task_def.name)
    self.assertEqual(final_count, 0)
```

**Example 2: Test deletion preserves completed instances**

```python
def test_deletion_preserves_completed_instances(self):
    """Test that deleting task definition preserves completed instances."""
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Delete Completed Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=due_date,
    )

    # Get an instance and mark it as completed
    instances = frappe.get_all(
        "CG Task Instance",
        filters={"task_definition_id": task_def.name},
        fields=["name"],
        limit=1,
    )

    if instances:
        instance = frappe.get_doc("CG Task Instance", instances[0].name)
        instance.is_completed = 1

        # Provide minimal request context to avoid errors
        try:
            frappe.local.request = type("Req", (), {"headers": {}})()
        except Exception:
            pass

        instance.save(ignore_permissions=True)

        completed_count = frappe.db.count(
            "CG Task Instance",
            {
                "task_definition_id": task_def.name,
                "is_completed": 1,
            },
        )

        # Delete the task definition
        frappe.delete_doc("CG Task Definition", task_def.name, force=1, ignore_permissions=True)

        # Verify completed instance still exists
        final_completed_count = frappe.db.count(
            "CG Task Instance",
            {
                "task_definition_id": task_def.name,
                "is_completed": 1,
            },
        )

        self.assertEqual(final_completed_count, completed_count)
```

**Key takeaway**: You almost never construct `CG Task Definition` manually in tests; you use `create_task_definition` as a high‑level API that hides child‑table wiring and permissions.

---

## 3. CG Task Instance tests – Complete Examples

File: `clapgrow_app/clapgrow_app/doctype/cg_task_instance/test_cg_task_instance.py`

### 3.1 Test Class Structure

**Example: Complete test class setup**

```python
import frappe
from frappe.tests.utils import FrappeTestCase

from clapgrow_app.clapgrow_app.doctype.cg_task_instance import test_status_priority_map
from clapgrow_app.tests.utils import create_cg_user, create_task_instance


class TestCGTaskInstance(FrappeTestCase):
    def setUp(self):
        super().setUp()
        # Set constants
        self.company_id = "Clapgrow Technology Private Ltd."
        self.department = "Test Department"
        self.branch = "Test Branch"

        # Create users
        self.assignee = self.create_assignee()
        self.member_1 = self.create_cg_member()

        # Set current user
        if self.assignee:
            frappe.set_user(self.assignee)

    def tearDown(self):
        frappe.db.rollback()

    def create_assignee(self):
        """Create an assignee user."""
        assignee = create_cg_user(
            first_name="assignee",
            email="assignee@test.cg",
            phone="+91 9876543210"
        )
        if assignee:
            return assignee.name

    def create_cg_member(self):
        """Create a member user."""
        member = create_cg_user(
            first_name="member1",
            email="member1@test.cg",
            phone="+91 9999999991",
            role="ROLE-Member"
        )
        if member:
            return member.name

    def create_task(self, task_name: str, due_at, priority: str):
        """Helper to create a task with consistent setup."""
        doc = create_task_instance(
            task_name,
            assigned_to=self.member_1 or frappe.session.user,
            assignee=self.assignee or frappe.session.user,
            due_date=due_at,
            priority=priority,
        )
        doc.reload()
        return doc
```

---

### 3.2 Using test_status_priority_map Helper Module

**File**: `clapgrow_app/clapgrow_app/doctype/cg_task_instance/test_status_priority_map.py`

**Available helper functions**:

1. **`expected_prefix(status, priority)`** - Computes expected 4-char prefix
2. **`assert_prefix(testcase, doc, status, priority)`** - Assertion helper
3. **`make_due_today_pair(base=None)`** - Returns two timestamps (early & late) for same-day ordering
4. **`force_scheduler_overdue_and_refresh(doc)`** - Forces scheduler run and reloads doc

**Example: Using the helper module**

```python
from clapgrow_app.clapgrow_app.doctype.cg_task_instance import test_status_priority_map

# In your test:
task = self.create_task("Test Task", past_date, "Critical")
test_status_priority_map.assert_prefix(self, task, "Overdue", "Critical")
```

---

### 3.3 Test Examples

**Example 1: Test basic task creation and defaults**

```python
def test_validate_onetime_task_creation(self):
    """Ensure a one-time task can be created with minimal fields."""
    cg_task_instance = create_task_instance(
        "One time task",
        assigned_to=self.member_1,
        assignee=self.assignee,
    )

    # Assert defaults are set correctly
    self.assertEqual(cg_task_instance.task_type, "Onetime")
    self.assertEqual(cg_task_instance.enabled, 1)
    self.assertEqual(cg_task_instance.priority, "Medium")
    self.assertEqual(cg_task_instance.assigned_to, self.member_1)
    self.assertEqual(cg_task_instance.assignee, self.assignee)
```

**Example 2: Test status and priority maps with padding**

```python
def test_status_and_priority_maps_with_padding(self):
    """Test that status_priority_map is correctly formatted."""
    today = frappe.utils.add_to_date(
        frappe.utils.now_datetime(),
        minutes=1,
        as_datetime=True
    )
    past = frappe.utils.add_days(today, -1)

    # Due Today + Critical
    due_today_critical_task = self.create_task(
        "Testing Due Today + Critical",
        today,
        "Critical"
    )

    # Overdue + Critical
    overdue_critical_task = self.create_task(
        "Testing Overdue + Critical",
        past,
        "Critical"
    )

    # Assert status
    self.assertIn(due_today_critical_task.status, ("Due Today"))
    self.assertEqual(overdue_critical_task.status, "Overdue")

    # Assert maps are integers
    self.assertIsInstance(due_today_critical_task.status_map, int)
    self.assertIsInstance(due_today_critical_task.priority_map, int)
    self.assertEqual(due_today_critical_task.priority_map, 0)  # Critical = 0

    # Assert status_priority_map format (14 chars total)
    self.assertEqual(len(due_today_critical_task.status_priority_map), 14)

    # Use helper to assert prefix
    test_status_priority_map.assert_prefix(
        self,
        overdue_critical_task,
        "Overdue",
        "Critical"
    )
```

**Example 3: Test due date tiebreaker (same status/priority, different times)**

```python
def test_due_date_tiebreaker(self):
    """Test that tasks with same status/priority are ordered by due_date."""
    # Get two timestamps on the same day (early and late)
    early_due, later_due = test_status_priority_map.make_due_today_pair()

    earliest_task = self.create_task("Testing Tie A", early_due, "Critical")
    later_task = self.create_task("Testing Tie B", later_due, "Critical")

    # Same status and priority maps
    self.assertEqual(earliest_task.status_map, later_task.status_map)
    self.assertEqual(earliest_task.priority_map, later_task.priority_map)

    # But different status_priority_map (due to due_date difference)
    self.assertLess(
        earliest_task.status_priority_map,
        later_task.status_priority_map
    )
    self.assertNotEqual(
        earliest_task.status_priority_map,
        later_task.status_priority_map
    )
```

**Example 4: Test status precedence over priority**

```python
def test_status_precedence_over_priority(self):
    """Test that status takes precedence over priority in ordering."""
    base = frappe.utils.now_datetime()
    future_today, _ = test_status_priority_map.make_due_today_pair(base)
    past = frappe.utils.add_days(base, -1)

    # Due Today + Low priority
    due_today_low = self.create_task(
        "Status precedence - Due Today Low",
        future_today,
        "Low"
    )

    # Overdue + Critical priority
    overdue_critical = self.create_task(
        "Status precedence - Overdue Critical",
        past,
        "Critical"
    )

    # Assert prefixes
    test_status_priority_map.assert_prefix(
        self, due_today_low, "Due Today", "Low"
    )
    test_status_priority_map.assert_prefix(
        self, overdue_critical, "Overdue", "Critical"
    )

    # Due Today (even with Low) should come before Overdue (even with Critical)
    self.assertLess(
        due_today_low.status_priority_map[:4],
        overdue_critical.status_priority_map[:4]
    )
```

**Example 5: Test priority precedence within same status**

```python
def test_priority_precedence_within_same_status(self):
    """Test that priority matters when status is the same."""
    base = frappe.utils.now_datetime()
    due_today_critical_time, due_today_medium_time = (
        test_status_priority_map.make_due_today_pair(base)
    )

    due_today_critical = self.create_task(
        "Priority precedence - Due Today Critical",
        due_today_critical_time,
        "Critical"
    )
    due_today_medium = self.create_task(
        "Priority precedence - Due Today Medium",
        due_today_medium_time,
        "Medium"
    )

    test_status_priority_map.assert_prefix(
        self, due_today_critical, "Due Today", "Critical"
    )
    test_status_priority_map.assert_prefix(
        self, due_today_medium, "Due Today", "Medium"
    )

    # Critical should come before Medium (same status)
    self.assertLess(
        due_today_critical.status_priority_map[:4],
        due_today_medium.status_priority_map[:4]
    )
```

**Example 6: Test scheduler updates status_priority_map**

```python
def test_scheduler_updates_status_priority_map(self):
    """Test that scheduler correctly updates status when task becomes overdue."""
    past = frappe.utils.add_days(frappe.utils.now_datetime(), -2)
    task = self.create_task("Testing Scheduler", past, "Medium")

    # Force scheduler to run and update status
    test_status_priority_map.force_scheduler_overdue_and_refresh(task)

    # Task should now be Overdue
    self.assertEqual(task.status, "Overdue")

    # Verify status_priority_map prefix
    # Overdue + Medium = 0101 (status=01, priority=01)
    self.assertFalse(task.status_priority_map.startswith("0001"))  # Not Upcoming
    self.assertFalse(task.status_priority_map.startswith("0201"))  # Not Due Today
    self.assertFalse(task.status_priority_map.startswith("0301"))  # Not Completed
    self.assertTrue(task.status_priority_map.startswith("0101"))   # Overdue + Medium
```

**Example 7: Test task completion workflow**

```python
def test_task_completion_workflow(self):
    """Test marking a due-today task as completed and priority map update."""
    future_date = frappe.utils.add_to_date(
        frappe.utils.now_datetime(),
        minutes=1,
        as_datetime=True
    )
    task = self.create_task("Workflow test", future_date, "Medium")

    # Safely complete the task (avoid request header dependency)
    previous_get_request_header = getattr(frappe, "get_request_header", None)
    try:
        frappe.get_request_header = lambda *args, **kwargs: ""
        task.is_completed = 1
        task.save()
    finally:
        if previous_get_request_header is not None:
            frappe.get_request_header = previous_get_request_header

    task.reload()
    self.assertEqual(task.status, "Completed")

    # Completed + Medium → prefix 0301
    test_status_priority_map.assert_prefix(
        self, task, "Completed", "Medium"
    )
```

**Example 8: Test validation with restrict flag**

```python
def test_invalid_data_validation(self):
    """Test validation that restrict prevents early completion."""
    # Create a future-due task with restrict=1
    future_date = frappe.utils.add_to_date(
        frappe.utils.now_datetime(),
        days=2,
        as_datetime=True
    )
    task = self.create_task("Restrict validation", future_date, "Low")
    task.restrict = 1

    # Completing should error
    with self.assertRaises(frappe.ValidationError):
        previous_get_request_header = getattr(frappe, "get_request_header", None)
        try:
            frappe.get_request_header = lambda *args, **kwargs: ""
            task.is_completed = 1
            task.save()
        finally:
            if previous_get_request_header is not None:
                frappe.get_request_header = previous_get_request_header
```

**Key takeaway**: CG task instance tests use:

- `create_cg_user` / `create_task_instance` to create realistic data.
- `test_status_priority_map` for reusable assertions and date helpers.
- Class‑level helpers (`create_task`) to keep individual tests tiny and readable.

---

## 4. CG User tests – current state & how to extend

File: `clapgrow_app/clapgrow_app/doctype/cg_user/test_cg_user.py` currently only defines an empty `TestCGUser(FrappeTestCase)` class.

When you add real CG user tests, follow the same **pattern**:

1. **Import helpers**:

```python
from clapgrow_app.tests.utils import (
    create_cg_company,
    create_cg_department,
    create_cg_branch,
    create_cg_user,
)
```

2. **Use a `setUp` method** to build basic fixtures:
   - Create a company, department, branch, and a small set of users via the helpers.
   - Optionally set `frappe.set_user(...)` to simulate who is performing the action.

3. **Write focused tests** that:
   - Call CG User methods / validations.
   - Assert on behaviour, not on how the fixtures were constructed.

This keeps CG User tests consistent with the task definition / instance tests.

---

## 5. General best practices from these tests

- **Always use shared helpers from `tests/utils.py`** for:
  - Creating companies / departments / branches / users.
  - Creating task definitions and task instances.
- **Keep test classes responsible for**:
  - Combining helpers into meaningful fixtures (`setup_test_data`, `create_task`, etc.).
  - Choosing the current user (`frappe.set_user`).
  - Asserting business rules and side‑effects.
- **Roll back DB state between tests**:
  - Use `frappe.db.rollback()` in `tearDown` (as in existing tests) if you’re not relying on Frappe’s default behaviour.
- **Extract shared assertion / helper logic into separate modules**:
  - Example: `test_status_priority_map.py` used by multiple task instance tests.
  - This keeps each individual test method small and declarative.

---

## 6. When to use mocks (and when not to) - Complete Examples

In Frappe apps, you usually prefer **real DB + fixtures** over heavy mocking, because:

- You already have a test database and helpers that create realistic documents.
- Many behaviours depend on DB state, permissions, and hooks, which mocks can easily hide or break.

---

### 6.1 Use Mocks - Example Scenarios

#### Example 1: Mock external HTTP calls (WhatsApp API)

**Scenario**: Testing a function that sends WhatsApp messages via external API.

**❌ Without mock (would hit real API)**:

```python
def test_send_whatsapp_notification(self):
    """This would actually send a WhatsApp message!"""
    user = create_cg_user(email="test@example.com")
    send_whatsapp_message(user.phone, "Test message")
    # Problem: Actually sends message, costs money, slow
```

**✅ With mock (isolates external dependency)**:

```python
from unittest.mock import patch, MagicMock

def test_send_whatsapp_notification(self):
    """Test that WhatsApp API is called correctly without actually sending."""
    user = create_cg_user(email="test@example.com")

    # Mock the external HTTP client
    with patch('clapgrow_app.api.whatsapp.requests.post') as mock_post:
        mock_post.return_value.status_code = 200
        mock_post.return_value.json.return_value = {"success": True}

        # Call the function
        result = send_whatsapp_message(user.phone, "Test message")

        # Assert API was called correctly
        mock_post.assert_called_once()
        call_args = mock_post.call_args
        self.assertIn(user.phone, str(call_args))
        self.assertEqual(result["success"], True)
```

#### Example 2: Mock email sending

**Scenario**: Testing that email notifications are sent when task is completed.

**❌ Without mock (would send real emails)**:

```python
def test_task_completion_sends_email(self):
    """This would send real emails!"""
    task = create_task_instance(
        "Test task",
        assigned_to="user@example.com"
    )
    task.is_completed = 1
    task.save()  # Would trigger real email send
    # Problem: Spams real email addresses
```

**✅ With mock (asserts email is called, doesn't send)**:

```python
from unittest.mock import patch

def test_task_completion_sends_email(self):
    """Test that email is sent on completion without actually sending."""
    task = create_task_instance(
        "Test task",
        assigned_to="user@example.com"
    )

    # Mock frappe.sendmail
    with patch('frappe.sendmail') as mock_sendmail:
        task.is_completed = 1
        task.save()

        # Assert email was called
        mock_sendmail.assert_called_once()

        # Verify email content
        call_args = mock_sendmail.call_args
        self.assertIn("completed", call_args[1]["subject"].lower())
```

#### Example 3: Mock background job enqueueing

**Scenario**: Testing that a long-running job is enqueued (but not executed).

**❌ Without mock (would actually run the job)**:

```python
def test_process_large_dataset(self):
    """This would actually process thousands of records!"""
    process_large_dataset()  # Takes 10 minutes!
    # Problem: Too slow for unit tests
```

**✅ With mock (asserts job is enqueued, doesn't run)**:

```python
from unittest.mock import patch

def test_process_large_dataset_enqueues_job(self):
    """Test that job is enqueued without actually running it."""
    with patch('frappe.enqueue') as mock_enqueue:
        process_large_dataset()

        # Assert job was enqueued
        mock_enqueue.assert_called_once()

        # Verify job parameters
        call_args = mock_enqueue.call_args
        self.assertEqual(call_args[0][0], "clapgrow_app.api.tasks.process_dataset")
```

#### Example 4: Mock error conditions (timeout, 500 errors)

**Scenario**: Testing error handling when external service fails.

**✅ With mock (simulates failure)**:

```python
from unittest.mock import patch
import requests

def test_handles_api_timeout(self):
    """Test that timeout errors are handled gracefully."""
    with patch('requests.get') as mock_get:
        # Simulate timeout
        mock_get.side_effect = requests.Timeout("Connection timeout")

        # Should handle error gracefully
        result = fetch_external_data()
        self.assertIsNone(result)
        self.assertIn("timeout", str(result.error).lower())
```

#### Example 5: Mock for pure function unit test

**Scenario**: Testing a pure calculation function that uses `frappe.db.get_value`.

**✅ With mock (isolates pure logic)**:

```python
from unittest.mock import patch
from frappe.tests import UnitTestCase

class TestTaskPriorityCalculator(UnitTestCase):
    """Unit test for pure calculation logic."""

    @patch('frappe.db.get_value')
    def test_calculate_priority_score(self, mock_get_value):
        """Test priority calculation without hitting DB."""
        # Mock DB call
        mock_get_value.return_value = 5  # Mock task count

        # Test pure calculation
        score = calculate_priority_score("user@example.com")

        # Assert calculation is correct
        self.assertEqual(score, 25)  # 5 * 5 = 25

        # Verify DB was called
        mock_get_value.assert_called_once()
```

**Note**: You should also have an **integration test** that uses real DB:

```python
class TestTaskPriorityCalculatorIntegration(FrappeTestCase):
    """Integration test with real DB."""

    def test_calculate_priority_score_with_real_db(self):
        """Test priority calculation with real database."""
        user = create_cg_user(email="test@example.com")
        # Create real tasks
        create_task_instance("Task 1", assigned_to=user.name)
        create_task_instance("Task 2", assigned_to=user.name)

        # Test with real DB
        score = calculate_priority_score(user.name)
        self.assertEqual(score, 10)  # 2 tasks * 5 = 10
```

---

### 6.2 Avoid Mocks - Example Scenarios

#### Example 1: Testing DocType validation (NO MOCK)

**✅ Correct approach (real DB)**:

```python
def test_recurring_task_requires_due_date(self):
    """Test validation with real DocType."""
    with self.assertRaises(frappe.ValidationError):
        task_def = create_task_definition(
            task_name="Test Task",
            assigned_to=self.member_user.name,
            frequency="Daily",
            do_not_save=True,  # Use helper, but don't save
        )
        task_def.save()  # Real validation runs
```

**❌ Wrong approach (mocking would hide validation)**:

```python
# DON'T DO THIS - hides real validation logic
@patch('frappe.get_doc')
def test_recurring_task_requires_due_date(self, mock_get_doc):
    # Mock hides the actual validation
    # You're not testing real behavior
```

#### Example 2: Testing permissions (NO MOCK)

**✅ Correct approach (real users and roles)**:

```python
def test_member_can_only_assign_to_self(self):
    """Test permissions with real users and roles."""
    # Create real users with real roles
    member = create_cg_user(
        email="member@test.com",
        role="ROLE-Member"
    )

    # Disable real permissions
    frappe.db.set_value("CG Role", member.role, "assign_team_lead", 0)

    # Switch to member user
    frappe.set_user(member.email)

    # Test real permission check
    with self.assertRaises(frappe.ValidationError):
        create_task_definition(
            task_name="Test",
            assigned_to="teamlead@test.com",  # Should fail
            assignee=member.email,
            frequency="Daily",
            due_date=self.get_future_due_date(),
        )
```

**❌ Wrong approach (mocking permissions)**:

```python
# DON'T DO THIS - doesn't test real permission system
@patch('frappe.has_permission')
def test_member_can_only_assign_to_self(self, mock_has_permission):
    # Mock doesn't test actual permission logic
    # Real permission checks might be different
```

#### Example 3: Testing DB-backed behavior (NO MOCK)

**✅ Correct approach (real DB operations)**:

```python
def test_pause_deletes_future_instances(self):
    """Test that pausing deletes instances with real DB."""
    task_def = create_task_definition(
        task_name="Test Task",
        assigned_to=self.member_user.name,
        frequency="Daily",
        due_date=self.get_future_due_date(),
    )

    # Real DB: count instances
    initial_count = self.count_task_instances(task_def.name)
    self.assertGreater(initial_count, 0)

    # Real DB: pause task
    task_def.is_paused = 1
    task_def.pause_start_date = add_to_date(days=1)
    task_def.pause_end_date = add_to_date(days=3)
    task_def.save()

    # Real DB: verify instances deleted
    instances_in_pause = frappe.get_all(
        "CG Task Instance",
        filters={
            "task_definition_id": task_def.name,
            "due_date": ["between", [pause_start, pause_end]]
        }
    )
    self.assertEqual(len(instances_in_pause), 0)
```

**❌ Wrong approach (mocking DB)**:

```python
# DON'T DO THIS - doesn't test real DB behavior
@patch('frappe.db.get_all')
def test_pause_deletes_instances(self, mock_get_all):
    # Mock doesn't test actual DB operations
    # Real on_trash hooks, cascades, etc. won't run
```

#### Example 4: Testing scheduler updates (NO MOCK)

**✅ Correct approach (real scheduler)**:

```python
def test_scheduler_updates_status_priority_map(self):
    """Test scheduler with real status update logic."""
    past = frappe.utils.add_days(frappe.utils.now_datetime(), -2)
    task = self.create_task("Testing Scheduler", past, "Medium")

    # Real scheduler call
    from clapgrow_app.api.login import update_task_status
    update_task_status()

    task.reload()
    self.assertEqual(task.status, "Overdue")
```

**❌ Wrong approach (mocking scheduler)**:

```python
# DON'T DO THIS - doesn't test real scheduler logic
@patch('clapgrow_app.api.login.update_task_status')
def test_scheduler_updates_status(self, mock_update):
    # Mock doesn't test actual status calculation
    # Real business logic is bypassed
```

---

### 6.3 Decision Tree: Mock or Not?

**Use MOCK when:**

- ✅ External API calls (HTTP, email, SMS, WhatsApp)
- ✅ Long-running background jobs (test enqueueing, not execution)
- ✅ Hard-to-reproduce error conditions (timeouts, network failures)
- ✅ Pure functions with DB dependencies (unit test only, with integration test backup)

**DON'T MOCK when:**

- ❌ DocType validations (`validate`, `before_save`, etc.)
- ❌ Permission checks (use real users/roles)
- ❌ DB operations (create, update, delete, queries)
- ❌ Scheduler jobs (use real scheduler calls)
- ❌ Business logic that depends on DB state

**Rule of thumb:**

- **Unit test + mock** = Pure logic + external effects
- **Integration test, no mock** = DocTypes, queries, business flows

---

## 7. What the Frappe framework itself uses for testing

Frappe’s own test framework lives in the `frappe/tests` package. On your system you can see it under:

- `/home/clapgrow/training/apps/frappe/frappe/tests`

Key points from Frappe’s `frappe/tests/README.md`:

- **Test case classes**:
  - **`UnitTestCase`** (`from frappe.tests import UnitTestCase`)
    - Extends `unittest.TestCase`.
    - For **unit tests**: individual functions/classes in isolation.
    - Provides Frappe‑specific assertions and context managers (user switching, time freezing, etc.).
  - **`IntegrationTestCase`** (`from frappe.tests import IntegrationTestCase`)
    - Extends `UnitTestCase`.
    - For **integration tests** that require a real site + DB connection.
    - Handles:
      - Automatic site/DB setup.
      - Loading test records.
      - Resetting thread‑locals and context.
- **Utilities and generators**:
  - `frappe/tests/utils` contains generators and helpers for creating test data.
- **Core tests**:
  - Files like `test_api.py`, `test_document.py`, `test_db.py` use these base classes to test Frappe internals.

Your app (`clapgrow_app`) currently uses:

- `from frappe.tests.utils import FrappeTestCase`

This is historically similar to the framework’s integration‑style test cases: it sets up a Frappe environment and DB so you can hit real DocTypes and APIs in tests.

---

## 8. Unit tests vs integration tests - Complete Examples

In the context of Frappe and this app, understanding the difference is crucial.

---

### 8.1 Unit Tests - Examples

**Characteristics:**

- Focus on a **single function or class**
- Ideally do **not** hit the database or network
- Fast execution
- Can use mocks for dependencies

#### Example 1: Pure utility function (unit test)

**Code to test** (`clapgrow_app/utils.py`):

```python
def calculate_task_score(priority: str, days_overdue: int) -> int:
    """Calculate task priority score."""
    priority_weights = {"Critical": 10, "High": 5, "Medium": 2, "Low": 1}
    weight = priority_weights.get(priority, 1)
    return weight * (days_overdue + 1)
```

**✅ Unit test (no DB, no Frappe)**:

```python
from unittest import TestCase
from clapgrow_app.utils import calculate_task_score

class TestCalculateTaskScore(TestCase):
    """Unit test for pure calculation function."""

    def test_critical_priority_score(self):
        """Test critical priority calculation."""
        score = calculate_task_score("Critical", 0)
        self.assertEqual(score, 10)  # 10 * (0 + 1)

    def test_high_priority_with_overdue(self):
        """Test high priority with overdue days."""
        score = calculate_task_score("High", 3)
        self.assertEqual(score, 20)  # 5 * (3 + 1)

    def test_unknown_priority_defaults(self):
        """Test unknown priority uses default weight."""
        score = calculate_task_score("Unknown", 2)
        self.assertEqual(score, 3)  # 1 * (2 + 1)
```

#### Example 2: Date calculation function (unit test)

**Code to test**:

```python
from datetime import datetime, timedelta

def get_next_working_day(date: datetime, holidays: list) -> datetime:
    """Get next working day excluding holidays."""
    next_day = date + timedelta(days=1)
    while next_day.date() in holidays:
        next_day += timedelta(days=1)
    return next_day
```

**✅ Unit test (pure logic)**:

```python
from unittest import TestCase
from datetime import datetime
from clapgrow_app.utils import get_next_working_day

class TestGetNextWorkingDay(TestCase):
    """Unit test for date calculation."""

    def test_skips_holidays(self):
        """Test that holidays are skipped."""
        base_date = datetime(2024, 1, 1)  # Monday
        holidays = [datetime(2024, 1, 2).date()]  # Tuesday is holiday

        result = get_next_working_day(base_date, holidays)

        # Should skip Tuesday, return Wednesday
        self.assertEqual(result.date(), datetime(2024, 1, 3).date())

    def test_no_holidays_returns_next_day(self):
        """Test normal case with no holidays."""
        base_date = datetime(2024, 1, 1)
        holidays = []

        result = get_next_working_day(base_date, holidays)

        self.assertEqual(result.date(), datetime(2024, 1, 2).date())
```

#### Example 3: Status prefix calculation (unit test)

**Code to test** (from `test_status_priority_map.py`):

```python
def expected_prefix(status: str, priority: str) -> str:
    """Compute expected 4-char prefix using system maps."""
    status_map = STATUS_ORDER.get(status, 999)
    priority_map = PRIORITY_ORDER.get(priority, 999)
    return f"{status_map:02d}{priority_map:02d}"
```

**✅ Unit test (pure function)**:

```python
from unittest import TestCase
from clapgrow_app.clapgrow_app.doctype.cg_task_instance.test_status_priority_map import (
    expected_prefix
)

class TestExpectedPrefix(TestCase):
    """Unit test for prefix calculation."""

    def test_overdue_critical_prefix(self):
        """Test Overdue + Critical prefix."""
        prefix = expected_prefix("Overdue", "Critical")
        self.assertEqual(prefix, "0100")  # status=01, priority=00

    def test_due_today_medium_prefix(self):
        """Test Due Today + Medium prefix."""
        prefix = expected_prefix("Due Today", "Medium")
        self.assertEqual(prefix, "0201")  # status=02, priority=01

    def test_unknown_status_defaults(self):
        """Test unknown status uses default."""
        prefix = expected_prefix("Unknown", "Critical")
        self.assertEqual(prefix, "99900")  # status=999, priority=00
```

---

### 8.2 Integration Tests - Examples

**Characteristics:**

- Exercise **multiple layers together** (DocTypes + DB + hooks + permissions)
- Use real database
- Test real business flows
- Slower execution

#### Example 1: DocType validation (integration test)

**✅ Integration test (real DB, real DocType)**:

```python
from frappe.tests.utils import FrappeTestCase
from clapgrow_app.tests.utils import create_cg_user, create_task_definition

class TestTaskDefinitionValidation(FrappeTestCase):
    """Integration test for DocType validation."""

    def setUp(self):
        self.member_user = create_cg_user(
            email="member@test.com",
            role="ROLE-Member"
        )

    def test_recurring_task_requires_due_date(self):
        """Test validation with real DocType and DB."""
        with self.assertRaises(frappe.ValidationError):
            task_def = create_task_definition(
                task_name="Test Task",
                assigned_to=self.member_user.name,
                frequency="Daily",
                do_not_save=True,  # Real document, not saved yet
            )
            task_def.save()  # Real validation runs
```

#### Example 2: Permission system (integration test)

**✅ Integration test (real users, real roles, real permissions)**:

```python
class TestTaskAssignmentPermissions(FrappeTestCase):
    """Integration test for permission system."""

    def setUp(self):
        self.company = create_cg_company("Test Company")
        self.member = create_cg_user(
            email="member@test.com",
            role="ROLE-Member",
            company_id=self.company.name
        )
        self.team_lead = create_cg_user(
            email="teamlead@test.com",
            role="ROLE-Team Lead",
            company_id=self.company.name
        )

    def test_member_cannot_assign_to_team_lead(self):
        """Test real permission check."""
        # Disable permission
        frappe.db.set_value("CG Role", self.member.role, "assign_team_lead", 0)

        # Switch to member user
        frappe.set_user(self.member.email)

        # Real permission check should fail
        with self.assertRaises(frappe.ValidationError):
            create_task_definition(
                task_name="Test",
                assigned_to=self.team_lead.email,
                assignee=self.member.email,
                frequency="Daily",
                due_date=self.get_future_due_date(),
            )
```

#### Example 3: Scheduler + DB updates (integration test)

**✅ Integration test (real scheduler, real DB)**:

```python
class TestTaskStatusScheduler(FrappeTestCase):
    """Integration test for scheduler updates."""

    def test_scheduler_updates_overdue_tasks(self):
        """Test that scheduler updates task status in real DB."""
        # Create task with past due date
        past_date = frappe.utils.add_days(
            frappe.utils.now_datetime(),
            -2
        )
        task = create_task_instance(
            "Overdue Task",
            assigned_to="user@test.com",
            due_date=past_date
        )

        # Initial status
        self.assertEqual(task.status, "Upcoming")

        # Run real scheduler
        from clapgrow_app.api.login import update_task_status
        update_task_status()

        # Reload from real DB
        task.reload()

        # Verify real status update
        self.assertEqual(task.status, "Overdue")
        self.assertIsNotNone(task.status_priority_map)
```

#### Example 4: Full workflow (integration test)

**✅ Integration test (end-to-end workflow)**:

```python
class TestTaskWorkflow(FrappeTestCase):
    """Integration test for complete task workflow."""

    def setUp(self):
        self.company = create_cg_company("Test Company")
        self.user = create_cg_user(
            email="user@test.com",
            company_id=self.company.name
        )

    def test_complete_task_workflow(self):
        """Test complete workflow: create -> assign -> complete."""
        # 1. Create task definition
        task_def = create_task_definition(
            task_name="Daily Report",
            assigned_to=self.user.name,
            frequency="Daily",
            due_date=self.get_future_due_date()
        )

        # 2. Verify instances created in real DB
        instances = frappe.get_all(
            "CG Task Instance",
            filters={"task_definition_id": task_def.name}
        )
        self.assertGreater(len(instances), 0)

        # 3. Get first instance
        instance = frappe.get_doc("CG Task Instance", instances[0].name)

        # 4. Complete the task
        instance.is_completed = 1
        instance.save()

        # 5. Verify status updated in real DB
        instance.reload()
        self.assertEqual(instance.status, "Completed")
        self.assertEqual(instance.is_completed, 1)

        # 6. Verify task definition status updated
        task_def.reload()
        # (Depending on your logic, task_def status might update)
```

---

### 8.3 Side-by-Side Comparison

#### Scenario: Testing task priority calculation

**Unit Test Approach** (fast, isolated):

```python
from unittest import TestCase

class TestTaskPriorityCalculation(TestCase):
    """Unit test - pure logic only."""

    def test_calculate_priority(self):
        """Test calculation without DB."""
        from clapgrow_app.utils import calculate_priority

        result = calculate_priority("Critical", 5)
        self.assertEqual(result, 50)
```

**Integration Test Approach** (real DB, real workflow):

```python
from frappe.tests.utils import FrappeTestCase

class TestTaskPriorityIntegration(FrappeTestCase):
    """Integration test - real DB and workflow."""

    def test_task_priority_in_real_workflow(self):
        """Test priority calculation in real task creation."""
        user = create_cg_user(email="test@example.com")

        # Create real task
        task = create_task_instance(
            "Test Task",
            assigned_to=user.name,
            priority="Critical"
        )

        # Verify priority is set correctly in DB
        task.reload()
        self.assertEqual(task.priority, "Critical")
        self.assertEqual(task.priority_map, 0)  # Critical = 0
```

---

### 8.4 How to Choose Which to Write

**Write UNIT TEST when:**

- ✅ You have a **pure function** (no DB, no Frappe dependencies)
- ✅ You want to test **algorithm logic** in isolation
- ✅ You need **fast feedback** during development
- ✅ Function has **complex calculations** that need thorough testing

**Write INTEGRATION TEST when:**

- ✅ You're testing **DocType behavior** (validate, on_update, etc.)
- ✅ You're testing **permissions** (use real users/roles)
- ✅ You're testing **DB operations** (create, update, delete)
- ✅ You're testing **scheduler jobs** (real scheduler calls)
- ✅ You're testing **complete workflows** (end-to-end)

**Best Practice:**

- Write **both** when you have complex logic:
  - **Unit test** for the pure calculation/algorithm
  - **Integration test** to verify it works in the real Frappe environment

---

### 8.5 Complete Example: Both Unit and Integration Tests

**Code to test** (`clapgrow_app/utils.py`):

```python
def compute_task_sort_key(status: str, priority: str, due_date: datetime) -> str:
    """Compute sort key for task ordering."""
    status_map = STATUS_ORDER.get(status, 999)
    priority_map = PRIORITY_ORDER.get(priority, 999)
    due_timestamp = int(due_date.timestamp())
    return f"{status_map:02d}{priority_map:02d}{due_timestamp:014d}"
```

**Unit Test** (test pure logic):

```python
from unittest import TestCase
from datetime import datetime
from clapgrow_app.utils import compute_task_sort_key

class TestComputeTaskSortKey(TestCase):
    """Unit test for sort key calculation."""

    def test_sort_key_format(self):
        """Test that sort key has correct format."""
        due_date = datetime(2024, 1, 1, 12, 0, 0)
        key = compute_task_sort_key("Overdue", "Critical", due_date)

        # Format: status(2) + priority(2) + timestamp(14) = 18 chars
        self.assertEqual(len(key), 18)
        self.assertTrue(key.startswith("0100"))  # Overdue=01, Critical=00

    def test_sort_key_ordering(self):
        """Test that sort keys order correctly."""
        base_date = datetime(2024, 1, 1, 12, 0, 0)

        key1 = compute_task_sort_key("Overdue", "Critical", base_date)
        key2 = compute_task_sort_key("Due Today", "Critical", base_date)

        # Overdue should come before Due Today
        self.assertLess(key1, key2)
```

**Integration Test** (test in real environment):

```python
from frappe.tests.utils import FrappeTestCase
from clapgrow_app.tests.utils import create_cg_user, create_task_instance

class TestTaskSortKeyIntegration(FrappeTestCase):
    """Integration test for sort key in real tasks."""

    def test_task_has_correct_sort_key(self):
        """Test that real task has correct sort key."""
        user = create_cg_user(email="test@example.com")
        past_date = frappe.utils.add_days(
            frappe.utils.now_datetime(),
            -1
        )

        # Create real task
        task = create_task_instance(
            "Test Task",
            assigned_to=user.name,
            due_date=past_date,
            priority="Critical"
        )

        # Verify sort key is set correctly
        task.reload()
        self.assertIsNotNone(task.status_priority_map)
        self.assertEqual(len(task.status_priority_map), 14)

        # Verify it starts with correct prefix
        self.assertTrue(task.status_priority_map.startswith("0100"))
```

**Key takeaway**: Write both unit and integration tests for complex logic:

- **Unit test** = Fast, isolated, tests pure logic
- **Integration test** = Real environment, tests everything works together

---

## 9. Step-by-Step: Writing a New Test from Scratch

This section walks you through writing a complete test from scratch.

### 9.1 Scenario: Test that a task cannot be completed before its due date when `restrict=1`

**Step 1: Understand what you're testing**

- Feature: Tasks with `restrict=1` cannot be completed before due date
- Type: Integration test (needs real DocType, real validation)
- File: `clapgrow_app/clapgrow_app/doctype/cg_task_instance/test_cg_task_instance.py`

**Step 2: Set up the test class structure**

```python
import frappe
from frappe.tests.utils import FrappeTestCase
from frappe.utils import add_to_date, now_datetime

from clapgrow_app.tests.utils import create_cg_user, create_task_instance


class TestCGTaskInstance(FrappeTestCase):
    def setUp(self):
        super().setUp()
        # Create test users
        self.assignee = create_cg_user(
            first_name="Assignee",
            email="assignee@test.cg",
            phone="+91 9876543210"
        )
        self.member = create_cg_user(
            first_name="Member",
            email="member@test.cg",
            role="ROLE-Member"
        )
        if self.assignee:
            frappe.set_user(self.assignee.name)

    def tearDown(self):
        frappe.db.rollback()
```

**Step 3: Write the test method**

```python
def test_restrict_prevents_early_completion(self):
    """Test that restrict=1 prevents completion before due date."""
    # Step 3a: Create a task with future due date
    future_date = add_to_date(now_datetime(), days=2, as_datetime=True)

    task = create_task_instance(
        task_name="Restricted Task",
        assigned_to=self.member.name,
        assignee=self.assignee.name,
        due_date=future_date,
        priority="Medium"
    )

    # Step 3b: Set restrict flag
    task.restrict = 1
    task.save()
    task.reload()

    # Step 3c: Try to complete before due date (should fail)
    # Need to mock request header to avoid HTTP context issues
    previous_get_request_header = getattr(frappe, "get_request_header", None)
    try:
        frappe.get_request_header = lambda *args, **kwargs: ""
        task.is_completed = 1

        # Step 3d: Assert that save raises ValidationError
        with self.assertRaises(frappe.ValidationError) as context:
            task.save()

        # Step 3e: Verify error message
        self.assertIn("cannot be completed", str(context.exception).lower())
    finally:
        if previous_get_request_header is not None:
            frappe.get_request_header = previous_get_request_header
```

**Step 4: Run the test**

```bash
bench --site [your-site] run-tests --module clapgrow_app.clapgrow_app.doctype.cg_task_instance.test_cg_task_instance.TestCGTaskInstance.test_restrict_prevents_early_completion
```

**Complete test file example:**

```python
# Copyright (c) 2025, Clapgrow and Contributors
# See license.txt

import frappe
from frappe.tests.utils import FrappeTestCase
from frappe.utils import add_to_date, now_datetime

from clapgrow_app.tests.utils import create_cg_user, create_task_instance


class TestCGTaskInstance(FrappeTestCase):
    def setUp(self):
        super().setUp()
        self.assignee = create_cg_user(
            first_name="Assignee",
            email="assignee@test.cg",
            phone="+91 9876543210"
        )
        self.member = create_cg_user(
            first_name="Member",
            email="member@test.cg",
            role="ROLE-Member"
        )
        if self.assignee:
            frappe.set_user(self.assignee.name)

    def tearDown(self):
        frappe.db.rollback()

    def test_restrict_prevents_early_completion(self):
        """Test that restrict=1 prevents completion before due date."""
        future_date = add_to_date(now_datetime(), days=2, as_datetime=True)

        task = create_task_instance(
            task_name="Restricted Task",
            assigned_to=self.member.name,
            assignee=self.assignee.name,
            due_date=future_date,
            priority="Medium"
        )

        task.restrict = 1
        task.save()
        task.reload()

        previous_get_request_header = getattr(frappe, "get_request_header", None)
        try:
            frappe.get_request_header = lambda *args, **kwargs: ""
            task.is_completed = 1

            with self.assertRaises(frappe.ValidationError) as context:
                task.save()

            self.assertIn("cannot be completed", str(context.exception).lower())
        finally:
            if previous_get_request_header is not None:
                frappe.get_request_header = previous_get_request_header
```

---

### 9.2 Scenario: Test that weekly recurring tasks generate instances only on specified days

**Step 1: Understand what you're testing**

- Feature: Weekly tasks should only create instances on specified weekdays
- Type: Integration test (needs real DB, real instance generation)
- File: `clapgrow_app/clapgrow_app/doctype/cg_task_definition/test_cg_task_definition.py`

**Step 2: Use existing test class structure**

```python
# Already exists in TestCGTaskDefinition
# Just add a new test method
```

**Step 3: Write the test**

```python
def test_weekly_task_only_creates_on_specified_days(self):
    """Test that weekly tasks only create instances on specified weekdays."""
    # Step 3a: Create weekly task with specific days
    due_date = self.get_future_due_date()

    task_def = create_task_definition(
        task_name="Weekly Meeting",
        assigned_to=self.member_user.name,
        frequency="Weekly",
        due_date=due_date,
        week_days="Monday,Wednesday,Friday",  # Only these days
    )

    # Step 3b: Get all generated instances
    instances = frappe.get_all(
        "CG Task Instance",
        filters={"task_definition_id": task_def.name},
        fields=["name", "due_date"]
    )

    # Step 3c: Verify all instances are on correct weekdays
    from frappe.utils import get_datetime
    from clapgrow_app.clapgrow_app.doctype.cg_task_definition.cg_task_definition import WEEKDAYS

    allowed_days = ["Monday", "Wednesday", "Friday"]

    for instance in instances:
        due_dt = get_datetime(instance.due_date)
        weekday_name = WEEKDAYS[due_dt.weekday()]

        # Step 3d: Assert weekday is in allowed list
        self.assertIn(
            weekday_name,
            allowed_days,
            f"Instance {instance.name} is on {weekday_name}, not in {allowed_days}"
        )
```

---

### 9.3 Scenario: Test a pure utility function (unit test)

**Step 1: Understand what you're testing**

- Feature: A pure function that calculates task urgency score
- Type: Unit test (no DB, no Frappe)
- File: `clapgrow_app/tests/test_utils.py` (new file)

**Step 2: Create new test file**

```python
# clapgrow_app/tests/test_utils.py
from unittest import TestCase
from clapgrow_app.utils import calculate_urgency_score


class TestCalculateUrgencyScore(TestCase):
    """Unit tests for urgency score calculation."""

    def test_critical_overdue_has_high_score(self):
        """Test that critical overdue tasks have high urgency."""
        score = calculate_urgency_score("Critical", 5)  # 5 days overdue
        self.assertEqual(score, 50)  # 10 * 5

    def test_low_future_has_low_score(self):
        """Test that low priority future tasks have low urgency."""
        score = calculate_urgency_score("Low", -2)  # 2 days in future
        self.assertEqual(score, -2)  # 1 * -2
```

**Step 3: Run the test**

```bash
bench --site [your-site] run-tests --module clapgrow_app.tests.test_utils
```

---

### 9.4 Common Patterns Checklist

When writing a new test, ask yourself:

**✅ Setup:**

- [ ] Do I need to create users? → Use `create_cg_user`
- [ ] Do I need company/department/branch? → Use `create_cg_company`, etc.
- [ ] Do I need to set current user? → Use `frappe.set_user(user.email)`

**✅ Test Data:**

- [ ] Creating a task? → Use `create_task_instance` or `create_task_definition`
- [ ] Need specific dates? → Use `add_to_date`, `now_datetime`, or class helpers
- [ ] Need to avoid saving? → Use `do_not_save=True`

**✅ Assertions:**

- [ ] Testing validation errors? → Use `with self.assertRaises(frappe.ValidationError)`
- [ ] Testing DB state? → Use `frappe.get_all`, `frappe.db.count`, or `doc.reload()`
- [ ] Testing permissions? → Use real users and `frappe.set_user`

**✅ Cleanup:**

- [ ] Rollback DB? → Use `frappe.db.rollback()` in `tearDown`
- [ ] Restore mocked functions? → Use `try/finally` blocks

---

### 9.5 Debugging Failed Tests

**Problem: Test fails with permission error**

```python
# ❌ Problem
frappe.exceptions.PermissionError: You do not have permission to access CG Task Instance

# ✅ Solution: Use helper with ignore_permissions
task = create_task_instance(
    "Test",
    assigned_to=user.name
)  # Helper automatically uses ignore_permissions in test mode
```

**Problem: Test fails because user doesn't exist**

```python
# ❌ Problem
frappe.exceptions.DoesNotExistError: CG User test@example.com does not exist

# ✅ Solution: Create user first
user = create_cg_user(email="test@example.com")
task = create_task_instance("Test", assigned_to=user.name)
```

**Problem: Test fails because document not saved**

```python
# ❌ Problem
AttributeError: 'NoneType' object has no attribute 'name'

# ✅ Solution: Check if you used do_not_save=True
task = create_task_instance("Test", assigned_to=user.name, do_not_save=True)
# task.name is None! Either save it or don't access .name
```

**Problem: Test is flaky (sometimes passes, sometimes fails)**

```python
# ❌ Problem: Using fixed dates that might be in the past
due_date = datetime(2024, 1, 1)  # Might be in the past now!

# ✅ Solution: Use relative dates
due_date = add_to_date(now_datetime(), days=1)  # Always future
```

---

### 9.6 Best Practices Summary

1. **Always use helpers from `tests/utils.py`** - Don't create documents manually
2. **Set up test data in `setUp`** - Create users, companies, etc. once per test class
3. **Rollback in `tearDown`** - Keep tests isolated
4. **Use descriptive test names** - `test_restrict_prevents_early_completion` not `test_restrict`
5. **Test one thing per test method** - Don't test multiple behaviors in one test
6. **Use real DB for integration tests** - Don't mock DocType operations
7. **Use mocks only for external APIs** - Email, HTTP, etc.
8. **Write both unit and integration tests** - For complex logic, test both pure function and real environment
