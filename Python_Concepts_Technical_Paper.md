##  Technical Paper 

## 1. Function Arguments: `args` and `kwargs`

In Python, `*args` and `**kwargs` are used to handle variable numbers of arguments in functions, providing flexibility when the exact number of inputs is unknown.

- `*args` **(Positional Arguments)**: The `*args` syntax allows a function to accept any number of positional arguments as a tuple. It is useful when you want to pass multiple non-keyword arguments without explicitly defining them in the function signature. For example:

  ```python
  def sum_numbers(*args):
      return sum(args)
  print(sum_numbers(1, 2, 3))  # Output: 6
  ```

  Here, `*args` collects `1`, `2`, and `3` into a tuple, which is then summed.

- `**kwargs` **(Keyword Arguments)**: The `**kwargs` syntax allows a function to accept any number of keyword arguments as a dictionary. It is ideal for handling named parameters dynamically. For example:

  ```python
  def print_details(**kwargs):
      for key, value in kwargs.items():
          print(f"{key}: {value}")
  print_details(name="Alice", age=30)  # Output: name: Alice, age: 30
  ```

  In this case, `**kwargs` collects the key-value pairs into a dictionary.

Using `*args` and `**kwargs` together allows functions to handle both positional and keyword arguments, enhancing their adaptability.

## 2. Function vs. Method

In Python, both functions and methods are callable objects, but they differ in their context and usage:

- **Function**: A function is a standalone block of code defined using the `def` keyword, typically outside a class. It operates independently and is not tied to any object. For example:

  ```python
  def greet(name):
      return f"Hello, {name}!"
  print(greet("Bob"))  # Output: Hello, Bob!
  ```

- **Method**: A method is a function defined inside a class and is associated with an instance of that class (or the class itself for class/static methods). Methods typically take `self` as their first parameter to access instance attributes. For example:

  ```python
  class Person:
      def greet(self, name):
          return f"Hello, {name} from {self.__class__.__name__}!"
  person = Person()
  print(person.greet("Bob"))  # Output: Hello, Bob from Person!
  ```

The key difference lies in their scope: functions are global or module-level, while methods are bound to objects or classes, enabling object-oriented programming paradigms.

## 3. Decorators in Python

Decorators are a powerful feature in Python that allow modification or extension of a function’s or method’s behavior without altering its code directly. A decorator is a higher-order function that takes another function as input and returns a new function with added functionality.

For example, consider a decorator that logs the execution time of a function:

```python
import time

def timer_decorator(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.2f} seconds")
        return result
    return wrapper

@timer_decorator
def slow_function():
    time.sleep(1)
    return "Done"

print(slow_function())  # Output: slow_function took 1.01 seconds, Done
```

Here, the `@timer_decorator` syntax applies the `timer_decorator` to `slow_function`, wrapping it with timing logic. Decorators are commonly used for logging, authentication, and memoization.

## 4. List vs. Tuple

Lists and tuples are two fundamental sequence types in Python, but they differ in mutability and use cases:

- **List**: A list is a mutable, ordered collection of elements, defined using square brackets `[]`. Elements can be added, removed, or modified after creation. For example:

  ```python
  my_list = [1, 2, 3]
  my_list.append(4)  # Modifies list to [1, 2, 3, 4]
  ```

- **Tuple**: A tuple is an immutable, ordered collection of elements, defined using parentheses `()`. Once created, its elements cannot be changed. For example:

  ```python
  my_tuple = (1, 2, 3)
  # my_tuple.append(4)  # Error: tuples are immutable
  ```

**Key Differences**:

- **Mutability**: Lists are mutable; tuples are immutable.
- **Performance**: Tuples are generally faster due to their immutability, making them suitable for fixed data.
- **Use Cases**: Lists are used when data needs to be modified (e.g., dynamic collections), while tuples are used for fixed data or as dictionary keys (since they are hashable).

## 5. Slicing in Python (Lists)

Slicing is a technique in Python to extract a subset of elements from a sequence, such as a list, using the syntax `[start:stop:step]`. It is highly flexible and widely used for manipulating lists.

- **How It Works**:
  - `start`: The index where the slice begins (inclusive, default is 0).
  - `stop`: The index where the slice ends (exclusive, default is the end of the list).
  - `step`: The increment between indices (default is 1).

For example:

```python
my_list = [0, 1, 2, 3, 4, 5]
print(my_list[1:4])    # Output: [1, 2, 3]
print(my_list[:3])     # Output: [0, 1, 2]
print(my_list[::2])    # Output: [0, 2, 4]
print(my_list[::-1])   # Output: [5, 4, 3, 2, 1, 0]
```

Slicing creates a new list containing the selected elements. Negative indices can be used to count from the end, and a negative `step` reverses the sequence. Slicing is efficient and intuitive, making it a powerful tool for list manipulation.

## 6. Dictionaries in Python

A dictionary is an unordered, mutable collection of key-value pairs, defined using curly braces `{}` or the `dict()` constructor. Keys must be unique and hashable (e.g., strings, numbers, or tuples), while values can be of any type.

For example:

```python
my_dict = {"name": "Alice", "age": 30, "city": "New York"}
print(my_dict["name"])  # Output: Alice
my_dict["age"] = 31     # Updates value
my_dict["job"] = "Engineer"  # Adds new key-value pair
```

**Key Features**:

- **Access**: Values are accessed using keys (e.g., `my_dict["key"]`).
- **Mutability**: Dictionaries can be modified by adding, updating, or removing key-value pairs.
- **Methods**: Common methods include `get()` (safely retrieve values), `keys()`, `values()`, and `items()` for iteration.
- **Use Cases**: Dictionaries are ideal for storing structured data, such as configuration settings or mappings.

## 

## References

- Python Software Foundation. (2025). *Python Official Documentation*. Available at: https://docs.python.org/3/
- Lutz, M. (2013). *Learning Python*. O'Reilly Media.