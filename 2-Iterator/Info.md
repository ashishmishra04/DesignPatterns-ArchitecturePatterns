# **Iterator Pattern** 

---

# ✅ **Iterator Pattern — Definition**

**The Iterator Pattern provides a way to access the elements of a collection sequentially without exposing the underlying data structure (such as lists, arrays, trees, or complex custom collections).**

It **decouples** the traversal logic from the actual collection so the collection can change internally without affecting how clients loop through it.

---

# ✅ **Iterator Pattern — Sample Code (Custom Iterator in C#)**

Here is a clean C# example implementing:

* A **collection (`NameCollection`)**
* A **custom iterator (`NameIterator`)**

### **Iterator Interface**

```csharp
public interface IIterator<T>
{
    bool HasNext();
    T Next();
}
```

---

### **Collection Interface**

```csharp
public interface IAggregate<T>
{
    IIterator<T> CreateIterator();
}
```

---

### **Concrete Collection**

```csharp
public class NameCollection : IAggregate<string>
{
    private readonly List<string> _names = new List<string>();

    public void Add(string name)
    {
        _names.Add(name);
    }

    public IIterator<string> CreateIterator()
    {
        return new NameIterator(_names);
    }
}
```

---

### **Concrete Iterator**

```csharp
public class NameIterator : IIterator<string>
{
    private readonly List<string> _names;
    private int _index = 0;

    public NameIterator(List<string> names)
    {
        _names = names;
    }

    public bool HasNext()
    {
        return _index < _names.Count;
    }

    public string Next()
    {
        return _names[_index++];
    }
}
```

---

### **Usage**

```csharp
var collection = new NameCollection();
collection.Add("Ashish");
collection.Add("Mishra");
collection.Add("Krux");

var iterator = collection.CreateIterator();

while (iterator.HasNext())
{
    Console.WriteLine(iterator.Next());
}
```

---

# 🧩 **How C# Already Uses Iterator Pattern Internally**

C# uses this pattern through:

* `IEnumerable`
* `IEnumerator`
* `foreach`

Example:

```csharp
foreach (var item in myList)
{
    Console.WriteLine(item);
}
```

Behind the scenes, this is the **Iterator Pattern**.

---

# 🌍 **Real-World Example**

### **Scenario: A File Reader That Returns Lines One-by-One**

Suppose you’re building a log processing application.
You don’t want to load the entire file into memory — you want to read each line **one at a time**.

This is exactly where the Iterator Pattern helps.

```csharp
public class FileLineIterator : IIterator<string>
{
    private readonly StreamReader _reader;
    private string _currentLine;

    public FileLineIterator(string filePath)
    {
        _reader = new StreamReader(filePath);
    }

    public bool HasNext()
    {
        _currentLine = _reader.ReadLine();
        return _currentLine != null;
    }

    public string Next()
    {
        return _currentLine;
    }
}
```

This hides complexity:

* buffer management
* file pointer movement
* memory issues

Client code just reads **line by line**.

---

# 🎯 **Why Do We Need the Iterator Pattern?** (Interview Answer)

1. **Encapsulation of internal structure**
   The client does not need to know whether the collection is

   * a List
   * a Tree
   * a HashSet
   * a custom graph

2. **Uniform way of traversal**
   The same iteration logic works with **any** collection.

3. **Clean and maintainable code**
   Traversal logic is separated from business logic.

4. **Supports multiple traversal strategies**
   Example: forward, backward, filtered, breadth-first, depth-first.

5. **Prevents exposing underlying representation**
   Avoids direct manipulation of internal data → more secure and stable.

---

# ✔ Interview Short Answer (If They Want a One-Liner)

**The Iterator Pattern provides a standard way to traverse collections without exposing how the collection is internally structured. It improves encapsulation, allows different traversal strategies, and makes client code clean and consistent.**

---

