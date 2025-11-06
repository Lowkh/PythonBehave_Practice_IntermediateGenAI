# Python Behave BDD Practical - Intermediate Level

## Introduction

This intermediate-level practical builds upon beginner fundamentals and introduces advanced Behave concepts for building more sophisticated and maintainable test suites. You'll learn to organize tests effectively, handle complex scenarios, and integrate best practices used in production environments[21][52].

---

## Part 1: Advanced Feature Organization with Background

### Understanding Background Sections

The **Background** keyword allows you to define preconditions that run before each scenario in a feature file. This eliminates repetitive "Given" steps[18][69].

### Creating a Realistic Example

Create `features/bank_account.feature`:

```gherkin
Feature: Bank Account Operations
    As a customer
    I want to manage my bank account
    So that I can perform financial transactions

    Background:
        Given a bank system is initialized
        And the customer "John Doe" exists with ID "12345"
        And the customer has an account with balance "$1000.00"
        And the account is active

    Scenario: Withdraw from account with sufficient funds
        When the customer withdraws "$200.00"
        Then the withdrawal should succeed
        And the account balance should be "$800.00"
        And a transaction record should be created

    Scenario: Withdraw from account with insufficient funds
        When the customer attempts to withdraw "$1500.00"
        Then the withdrawal should fail
        And an error message should be displayed
        And the account balance should remain "$1000.00"

    Scenario: Check account balance
        When the customer checks their balance
        Then the system should display "$1000.00"
        And the balance should match the database record
```

**Benefits of Background:**
- Reduces duplication across scenarios
- Makes each scenario focus on what's unique to that test case[18]
- Improves readability and maintainability
- Background runs before each scenario, resetting state for isolation[69]

---

## Part 2: Advanced Tag Usage and Test Filtering

### Understanding Tags

Tags are labels that help organize and selectively run tests[52][54]. They provide flexible test execution without modifying code.

### Implementing Tag Strategy

Update `features/bank_account.feature` with tags:

```gherkin
@bank
@critical
Feature: Bank Account Operations

    Background:
        Given a bank system is initialized
        And the customer "John Doe" exists with ID "12345"
        And the customer has an account with balance "$1000.00"
        And the account is active

    @smoke
    @high-priority
    Scenario: Withdraw from account with sufficient funds
        When the customer withdraws "$200.00"
        Then the withdrawal should succeed
        And the account balance should be "$800.00"

    @regression
    @low-priority
    Scenario: Withdraw from account with insufficient funds
        When the customer attempts to withdraw "$1500.00"
        Then the withdrawal should fail
        And the account balance should remain "$1000.00"

    @slow
    @integration
    Scenario: Complex multi-step transaction
        When the customer performs three consecutive transactions
        Then all transactions should be recorded
        And the final balance should be correct
```

### Running Tests with Tags

```bash
# Run only smoke tests
behave --tags=@smoke

# Run critical tests that are not slow
behave --tags=@critical --tags=-@slow

# Run tests with either high-priority OR smoke tag
behave --tags=@high-priority,@smoke

# Run tests with high-priority AND critical
behave --tags=@high-priority --tags=@critical

# Exclude slow tests
behave --tags=-@slow

# Complex tag expressions
behave --tags="@critical and not @slow"
behave --tags="(@smoke or @regression) and @bank"
```

**Common Tag Patterns:**
- `@smoke`: Quick validation tests
- `@regression`: Full test suite
- `@integration`: Tests requiring external systems
- `@slow`: Long-running tests
- `@wip`: Work in progress (under development)
- `@xfail`: Expected failures
- `@skip`: Skip execution[57]

---

## Part 3: Environment.py Hooks for Setup and Teardown

### Understanding Hooks

Hooks execute code at specific points in the test lifecycle. They handle setup/teardown without cluttering feature files[21][22][13].

Create `features/environment.py`:

```python
import logging
from database import Database
from bank_system import BankSystem

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def before_all(context):
    """Execute before the entire test suite runs once"""
    logger.info("Initializing test suite")
    context.db = Database(host="localhost", database="test_bank")
    context.bank_system = BankSystem(context.db)
    context.db.connect()
    logger.info("Database connection established")

def after_all(context):
    """Execute after the entire test suite completes"""
    logger.info("Cleaning up test suite")
    context.db.cleanup_test_data()
    context.db.disconnect()
    logger.info("Database connection closed")

def before_feature(context, feature):
    """Execute before each feature file"""
    logger.info(f"Starting feature: {feature.name}")
    context.test_data = {}

def after_feature(context, feature):
    """Execute after each feature file completes"""
    logger.info(f"Completed feature: {feature.name}")
    context.test_data.clear()

def before_scenario(context, scenario):
    """Execute before each scenario"""
    logger.info(f"Starting scenario: {scenario.name}")
    # Skip scenarios with specific tags
    if "skip" in scenario.effective_tags:
        scenario.mark_skipped()
    # Initialize scenario-specific data
    context.scenario_data = {}

def after_scenario(context, scenario):
    """Execute after each scenario, regardless of pass/fail"""
    logger.info(f"Scenario status: {scenario.status}")
    # Cleanup resources
    if hasattr(context, 'browser'):
        context.browser.quit()

def before_step(context, step):
    """Execute before each step"""
    logger.debug(f"Executing step: {step.name}")

def after_step(context, step):
    """Execute after each step"""
    if step.status == "failed":
        logger.error(f"Step failed: {step.name}")
        # Capture screenshot or debug info here
```

**Hook Execution Order:**[56]

```
before_all()
    for each feature:
        before_feature()
            for each scenario:
                before_scenario()
                    for each step:
                        before_step()
                        [execute step]
                        after_step()
                after_scenario()
        after_feature()
after_all()
```

---

## Part 4: Step Matchers - Parse, cfparse, and Regex

### Parse Matcher (Default)

The parse matcher uses the `parse` module for simple, readable parameter extraction[21][53]:

```python
# features/steps/step_definitions.py
from behave import given, when, then, use_step_matcher

# Parse matcher is the default
use_step_matcher("parse")

@given('the customer has an account with balance "{amount}"')
def step_account_balance(context, amount):
    """Extract a quoted string parameter"""
    context.account_balance = float(amount.replace("$", "").replace(",", ""))

@when('the customer withdraws "{amount}"')
def step_withdraw_amount(context, amount):
    """Extract quoted currency amount"""
    withdraw_amount = float(amount.replace("$", "").replace(",", ""))
    context.transaction_amount = withdraw_amount

@then('the account balance should be "{amount}"')
def step_verify_balance(context, amount):
    """Verify balance with currency formatting"""
    expected_balance = float(amount.replace("$", "").replace(",", ""))
    assert context.account_balance == expected_balance, \
        f"Expected {expected_balance}, got {context.account_balance}"
```

**Parse Type Converters:**

```python
@given('the customer has an account with balance {amount:f}')
def step_account_balance_typed(context, amount):
    """Use type conversion (:f for float)"""
    context.account_balance = amount

@when('the customer makes {count:d} transactions')
def step_multiple_transactions(context, count):
    """Integer type conversion (:d)"""
    context.transaction_count = count
```

### cfparse Matcher with Cardinality

The cfparse matcher supports cardinality field syntax for handling lists and optional values[60]:

```python
from behave import use_step_matcher, given, when

use_step_matcher("cfparse")

@given('the customer has {accounts:w+}')
def step_multiple_accounts(context, accounts):
    """Extract list of words separated by commas
    Input: Given the customer has checking, savings, money-market
    accounts = ['checking', 'savings', 'money-market']"""
    context.accounts = accounts.split(", ")

@given('the customer has {amounts:n+} in their accounts')
def step_account_amounts(context, amounts):
    """Extract list of numbers
    Input: Given the customer has 1000, 2000, 5000 in their accounts
    amounts = [1000, 2000, 5000]"""
    context.amounts = amounts

@given('the customer {may_have:w?} a premium account')
def step_optional_premium(context, may_have):
    """Optional field (?). Returns None if not matched"""
    context.has_premium = may_have is not None
```

### Regular Expression Matcher

Use regex for complex patterns that parse cannot handle[55]:

```python
from behave import use_step_matcher, given, when
import re

use_step_matcher("re")

@given(r'the account (?P<status>active|inactive|frozen)')
def step_account_status(context, status):
    """Use regex for multiple alternatives
    Input: Given the account active
    or: Given the account frozen"""
    context.account_status = status

@when(r'I transfer (?P<amount>\d+\.?\d*) from (?P<source>\w+) to (?P<destination>\w+)')
def step_transfer_money(context, amount, source, destination):
    """Complex regex pattern with multiple groups
    Input: When I transfer 500.00 from savings to checking"""
    context.transfer_amount = float(amount)
    context.transfer_source = source
    context.transfer_destination = destination
```

**Choosing the Right Matcher:**
- **parse**: Simple cases, more readable, default choice[21]
- **cfparse**: Lists and optional parameters[60]
- **re**: Complex patterns, backward compatibility[55]

---

## Part 5: Data Tables in Step Definitions

### Basic Data Tables

Data tables provide structured test data directly in scenarios[59]:

```gherkin
Scenario: Create multiple customer accounts
    Given the following customers exist:
        | name       | email              | age |
        | John Doe   | john@example.com   | 30  |
        | Jane Smith | jane@example.com   | 28  |
        | Bob Wilson | bob@example.com    | 35  |
    When I process all customers
    Then all customers should be in the system
```

**Implementing Data Table Steps:**

```python
from behave import given, when, then

@given('the following customers exist')
def step_create_customers(context):
    """Process data table from feature file
    context.table contains rows with column headers"""
    context.customers = []
    
    for row in context.table:
        # Access by column name
        customer = {
            'name': row['name'],
            'email': row['email'],
            'age': int(row['age'])
        }
        context.customers.append(customer)
        print(f"Created customer: {customer['name']}")

@when('I process all customers')
def step_process_customers(context):
    """Iterate through stored customers"""
    for customer in context.customers:
        context.bank_system.create_customer(customer)

@then('all customers should be in the system')
def step_verify_customers(context):
    """Verify all customers were created"""
    for customer in context.customers:
        db_customer = context.db.get_customer(customer['email'])
        assert db_customer is not None, f"Customer {customer['name']} not found"
```

### Vertical Data Tables

Data tables can be organized vertically (key-value style)[59]:

```gherkin
Scenario: Create user profile
    Given the following user details:
        | field    | value           |
        | name     | John Doe        |
        | email    | john@example.com|
        | username | johndoe         |
        | password | secure123       |
    When I create the user profile
    Then the profile should be active
```

**Implementing Vertical Data Tables:**

```python
@given('the following user details')
def step_user_details(context):
    """Process vertical table (each row is a key-value pair)"""
    context.user_profile = {}
    
    for row in context.table:
        # First column is the key, second is the value
        key = row['field']
        value = row['value']
        context.user_profile[key] = value
```

---

## Part 6: Custom Type Converters

### Creating User-Defined Types

Custom type converters allow complex parameter parsing[63]:

```python
# features/steps/step_definitions.py
from behave import register_type
import parse
from datetime import datetime, date

# Simple type converter
def parse_date(text):
    """Convert text to date object
    Format: YYYY-MM-DD
    Usage: When I check transactions from {date_value:Date}"""
    return datetime.strptime(text, "%Y-%m-%d").date()

# Register the type converter
register_type(Date=parse_date)

# With pattern annotation
@parse.with_pattern(r"\d{4}-\d{2}-\d{2}")
def parse_iso_date(text):
    """ISO format date with regex pattern"""
    return datetime.strptime(text, "%Y-%m-%d").date()

register_type(ISODate=parse_iso_date)

# Complex type with custom logic
def parse_currency(text):
    """Parse currency values
    Accepts: $1000, €1000, £500, ¥1000"""
    import re
    currency_pattern = r"([$€£¥])([0-9,]+\.?\d{0,2})"
    match = re.match(currency_pattern, text)
    
    if match:
        symbol, amount = match.groups()
        amount_value = float(amount.replace(",", ""))
        return {'symbol': symbol, 'amount': amount_value}
    raise ValueError(f"Invalid currency format: {text}")

register_type(Currency=parse_currency)

# Using custom types in steps
@given('the account balance is {balance:Currency}')
def step_set_balance(context, balance):
    context.currency_symbol = balance['symbol']
    context.balance = balance['amount']

@when('time is {transaction_date:Date}')
def step_set_transaction_date(context, transaction_date):
    context.transaction_date = transaction_date
```

---

## Part 7: Advanced Assertions with assertpy

### Installing assertpy

```bash
pip install assertpy
```

### Using assertpy for Fluent Assertions

Replace simple asserts with expressive, chainable assertions[66]:

```python
from assertpy import assert_that
from behave import then

@then('the account balance should be "{amount}"')
def step_verify_balance_assertpy(context, amount):
    """Use assertpy for readable assertions"""
    expected = float(amount.replace("$", ""))
    
    # Chainable assertions
    assert_that(context.account_balance) \
        .is_equal_to(expected) \
        .is_not_none() \
        .is_greater_than(0)

@then('the transaction should be recorded')
def step_verify_transaction(context):
    """Use assertpy for complex verifications"""
    assert_that(context.transactions).is_not_empty()
    
    # Find the transaction
    latest_transaction = context.transactions[-1]
    
    assert_that(latest_transaction) \
        .contains('amount', 'date', 'type') \
        .does_not_contain('error')

@then('customer list should contain {count:d} customers')
def step_verify_customer_count(context, count):
    """Chainable list assertions"""
    assert_that(context.customers) \
        .is_instance_of(list) \
        .is_length(count) \
        .is_not_empty()

@then('all transactions should have valid amounts')
def step_verify_transactions_valid(context):
    """Assertions on collections"""
    assert_that(context.transactions).extracting('amount') \
        .does_not_contain(None) \
        .does_not_contain(0) \
        .does_not_contain(-1)
```

**Common assertpy Methods:**
```python
# Equality and comparison
assert_that(value).is_equal_to(expected)
assert_that(value).is_not_equal_to(unexpected)
assert_that(value).is_greater_than(5)
assert_that(value).is_less_than(10)

# Type checking
assert_that(value).is_instance_of(str)
assert_that(value).is_type_of(dict)

# Collections
assert_that(items).is_length(3)
assert_that(items).contains('item')
assert_that(items).does_not_contain('missing')
assert_that(items).is_empty()
assert_that(items).is_not_empty()

# Strings
assert_that('text').starts_with('te')
assert_that('text').ends_with('xt')
assert_that('text').contains('ex')
assert_that('TEXT').is_upper()
```

---

## Part 8: Testing with Database Integration

### Setting Up Database Fixtures in environment.py

```python
# features/environment.py
from database import Database
import logging

logger = logging.getLogger(__name__)

class TestDatabase:
    def __init__(self, connection_string):
        self.db = Database(connection_string)
        self.test_data = {}
    
    def setup(self):
        self.db.connect()
        self.db.create_test_schema()
        self.db.seed_base_data()
    
    def teardown(self):
        self.db.drop_test_schema()
        self.db.disconnect()
    
    def cleanup_scenario(self):
        """Clean up data created during scenario"""
        for table, ids in self.test_data.items():
            for id_value in ids:
                self.db.delete_record(table, id_value)
        self.test_data.clear()

def before_all(context):
    """Initialize test database"""
    context.test_db = TestDatabase("postgresql://user:pass@localhost/test_db")
    context.test_db.setup()
    logger.info("Test database initialized")

def after_all(context):
    """Clean up test database"""
    context.test_db.teardown()
    logger.info("Test database cleaned up")

def after_scenario(context, scenario):
    """Clean up scenario-specific data"""
    context.test_db.cleanup_scenario()
```

### Using Database in Steps

```python
from behave import given, when, then

@given('a customer with ID {customer_id:d} exists in database')
def step_customer_exists(context, customer_id):
    """Verify customer exists in database"""
    customer = context.test_db.db.query(
        "SELECT * FROM customers WHERE id = %s", 
        (customer_id,)
    )
    assert customer, f"Customer {customer_id} not found"
    context.current_customer = customer[0]

@when('I create a new transaction')
def step_create_transaction(context):
    """Create transaction in database"""
    transaction = {
        'customer_id': context.current_customer['id'],
        'amount': context.transfer_amount,
        'type': 'transfer'
    }
    
    transaction_id = context.test_db.db.insert(
        'transactions', 
        transaction
    )
    
    # Track for cleanup
    if 'transactions' not in context.test_db.test_data:
        context.test_db.test_data['transactions'] = []
    context.test_db.test_data['transactions'].append(transaction_id)
    
    context.latest_transaction_id = transaction_id

@then('the transaction should be recorded in database')
def step_verify_transaction_recorded(context):
    """Verify transaction in database"""
    transaction = context.test_db.db.query(
        "SELECT * FROM transactions WHERE id = %s",
        (context.latest_transaction_id,)
    )
    assert transaction, "Transaction not found in database"
```

---

## Part 9: Organizing Step Definitions for Large Projects

### Project Structure for Scale

```
features/
├── environment.py
├── steps/
│   ├── __init__.py
│   ├── common_steps.py          # Shared Given steps
│   ├── account_steps.py          # Account-specific steps
│   ├── transaction_steps.py      # Transaction steps
│   ├── user_steps.py            # User management steps
│   └── utilities.py             # Helper functions
├── pages/                        # Page Object Models for UI testing
│   ├── base_page.py
│   ├── login_page.py
│   └── account_page.py
├── data/
│   ├── test_customers.json
│   └── test_accounts.json
├── account.feature
├── transactions.feature
└── users.feature
```

### Shared Utilities Module

```python
# features/steps/utilities.py
import logging

logger = logging.getLogger(__name__)

def create_test_customer(context, name, email):
    """Reusable helper function"""
    customer = {
        'name': name,
        'email': email,
        'created_at': datetime.now()
    }
    customer_id = context.test_db.db.insert('customers', customer)
    return customer_id

def verify_transaction_details(context, transaction_id, expected_fields):
    """Common verification logic"""
    transaction = context.test_db.db.query(
        "SELECT * FROM transactions WHERE id = %s",
        (transaction_id,)
    )
    
    for field, expected_value in expected_fields.items():
        actual_value = transaction[0].get(field)
        assert actual_value == expected_value, \
            f"Field {field}: expected {expected_value}, got {actual_value}"
    
    return transaction[0]
```

### Importing Across Modules

```python
# features/steps/account_steps.py
from behave import given, when, then
from utilities import create_test_customer, verify_transaction_details

@given('a test customer "{name}" with email "{email}"')
def step_create_customer(context, name, email):
    """Use shared utility function"""
    context.customer_id = create_test_customer(context, name, email)

@then('transaction {trans_id:d} should have the correct details')
def step_verify_transaction(context, trans_id):
    """Use shared verification function"""
    expected_fields = {
        'amount': context.expected_amount,
        'status': 'completed'
    }
    verify_transaction_details(context, trans_id, expected_fields)
```

---

## Part 10: Running Tests with Advanced Options

### Command-Line Options

```bash
# Run with detailed output
behave -v features/

# Stop on first failure
behave --stop

# Generate HTML report
behave -f html -o report.html

# Run specific feature file
behave features/account.feature

# Run specific scenario by line number
behave features/account.feature:25

# Run with tags
behave --tags=@critical --tags=-@slow

# Parallel execution (requires parallel plugin)
behave -f progress2 -k

# Dry run (don't execute, just check syntax)
behave --dry-run --no-capture

# Show undefined steps
behave features/ --dry-run --no-capture
```

### Creating behave.ini Configuration

```ini
# behave.ini
[behave]
# Path to features directory
paths = features

# Default output format
format = progress2

# Steps definition discovery
steps_dir = features/steps

# Environment module
environment.py = features/environment.py

# Tags to run by default
tags = -@skip

# Parallel workers
parallel_workers = 4
```

---

## Part 11: Intermediate Exercise

### Create a Complete E-Commerce Test Suite

1. **Create feature file** `features/shopping_cart.feature`:

```gherkin
@ecommerce
@critical
Feature: Shopping Cart Management
    
    Background:
        Given the product catalog is loaded
        And a new shopping cart is created
        And the customer "Alice" is logged in

    @smoke
    @high-priority
    Scenario Outline: Add products to cart
        When the customer adds <product> to cart with quantity <qty>
        Then the cart should contain <product>
        And the cart total should reflect the price

        Examples:
            | product    | qty |
            | Laptop     | 1   |
            | Mouse      | 2   |
            | Keyboard   | 1   |
```

2. **Implement steps** with data tables and database integration
3. **Use tags** to organize test execution
4. **Create environment.py** with proper setup/teardown
5. **Add assertions** using assertpy for clarity

---

## Part 12: Troubleshooting Advanced Scenarios

**Problem: Background steps not running**
- Solution: Ensure background is before any scenarios in feature file[69]

**Problem: Tags not working as expected**
- Solution: Verify tag syntax. Tags must start with @ and use correct boolean logic[57]

**Problem: Database state not clean between scenarios**
- Solution: Ensure after_scenario hook cleans up test data[21]

**Problem: Steps showing as undefined with complex matchers**
- Solution: Verify the correct step matcher is selected (parse/cfparse/re)[21]

**Problem: Data table rows missing first row**
- Solution: This is expected behavior. Data tables with headers strip the header row[62]

---

## Summary: Key Intermediate Concepts

✅ Background sections for reducing duplication[18][69]
✅ Tag-based test organization and filtering[52][54][57]
✅ Comprehensive hook lifecycle management[21][22][13][56]
✅ Multiple step matcher strategies (parse, cfparse, regex)[21][53][55]
✅ Data table processing and organization[59]
✅ Custom type converters for complex parsing[63]
✅ Advanced assertion libraries (assertpy)[66]
✅ Database integration and test data management
✅ Project organization for scalability
✅ Advanced command-line usage and configuration

---

## Next Steps to Advanced Level

1. **Page Object Model**: Organize web automation with reusable page classes[67]
2. **Parallel Execution**: Run tests concurrently for faster feedback
3. **API Testing**: Extend Behave for REST API and service testing
4. **Performance Testing**: Incorporate performance metrics into scenarios
5. **CI/CD Integration**: Integrate Behave with GitHub Actions, GitLab CI, or Jenkins
6. **Custom Formatters**: Create custom report formats
7. **Browser Automation**: Integrate Selenium WebDriver for web UI testing
8. **Behavior-Driven Development Practices**: Work with non-technical stakeholders

---

## References and Resources

- Official Behave Documentation: https://behave.readthedocs.io/
- Behave Examples Repository: https://github.com/behave/behave
- assertpy Documentation: https://assertpy.github.io/
- Parse Module Documentation: https://pypi.org/project/parse/
- Best Practices: https://automationpanda.com/2018/05/11/python-testing-101-behave/

Happy Advanced Testing! 🚀
