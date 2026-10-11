---
title: "Dependency Inversion in C#"
description: "Good habits in C#"
date: 2022-10-14T00:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, OOP, Csharp, programming]
---

Dependency inversion is an important principle for writing maintainable code. In this article we look at a C# example of applying it.

# Introduction and definition

Dependency inversion is one of the principles of the SOLID approach. It deals with the coupling between the different classes or modules of your application.
Dependency inversion involves the notions of higher-level and lower-level classes. A higher-level class depends on one or more lower-level classes. The core motto of dependency inversion is **every higher-level class should always depend on the abstraction of the lower-level class rather than on its concrete implementation**.

Stating the principle through its motto is not enough to fully grasp what dependency inversion actually means. So let's write some C# code to see this principle in action.

# Example

To better understand the dependency inversion principle, it seems relevant to study a practical example.
Suppose we have a main (higher-level) class named `TradeProcessor`. This class reads data from a source, then processes the raw data and stores it somewhere.

It depends on the following three classes:
- `DataProvider`, which provides the class with a set of methods to retrieve the raw data.
- `DataParser`, a class that contains methods to process the data and make it ready to store.
- `DataStorage`, a class responsible for storing the data in a database and writing logs.

## Implementation without dependency inversion

### The `TradeProcessor` class

```cs
namespace DependencyInversionInjection {

    public class TradeProcessor {

        private readonly DataProvider _data;
        private readonly DataParser _parser;
        private readonly DataStorage _storage;

        public TradeProcessor(DataProvider data, DataParser parser, DataStorage storage) 
        {
            _data = data;
            _parser = parser;
            _storage = storage;
        }

        public void ProcessTrade() {
            var lines = _data.GetAll();
            var trades = _parser.Parse(lines);
            _storage.Persist(trades);
        }
    }
}
```

### The `DataProvider` class

```cs
public class DataProvider
    {
        private readonly StreamReader _reader;

        public DataProvider(string path)
        {
            _reader = new StreamReader(path);
        }

        public List<string> GetAll()
        {
             var rawData = new List<string>();
            while(_reader.ReadLine() != null)
            {
                rawData.Add(_reader.ReadLine());
            }

            return rawData;
        }
    }
```

### The `DataParser` class

```cs
public class DataParser
    {

        public List<UserData> Parse(List<string> data)
        {
            string[] currentData;
            UserData userData;
            List<UserData> result = new List<UserData>();
            foreach(string line in data)
            {
                currentData = line.Split('/');
                userData = new UserData();
                userData.ID = Guid.Parse(currentData[0]);
                userData.Name = currentData[1];
                userData.PhoneNumber = currentData[2];

                result.Add(userData);
            }
            return result;
        }
    }

    public class UserData
    {
        public Guid ID { get; set; }
        public string Name { get; set; }
        public string PhoneNumber { get; set; }
    }
```

### The `DataStorage` class

```cs
    public class DataStorage
    {
        public void Persist(List<UserData> users)
        {
            var json = JsonSerializer.Serialize(users);
            File.WriteAllText("cleaned.json", json);
        }
    }
```

## Implementation with dependency inversion

The idea here is to make the `TradeProcessor` class depend not on the lower-level classes themselves, but on abstractions of those classes.
The element we use for abstraction is precisely an interface.

```cs
    public class TradeProcessorDI
    {
        private readonly IDataProvider _data;
        private readonly IDataParser _parser;
        private readonly IDataStorage _storage;
        public TradeProcessorDI(IDataProvider data, IDataParser parser, IDataStorage storage)
        {
            _data = data;
            _parser = parser;
            _storage = storage;
        }

        public void ProcessTrade()
        {
            var lines = _data.GetAll();
            var trades = _parser.Parse(lines);
            _storage.Persist(trades);
        }
    }
```

The `Main` method of our program works exactly the same with the two classes `TradeDataProcessor` and `TradeDataProcessorDI`.

```cs
    public class Program
    {
        static void Main(string[] args)
        {
            var data = new DataProvider("data/rawData.txt");
            var parser = new DataParser();
            var storage = new DataStorage();

            var processor = new TradeDataProcessorDI(data, parser, storage);
            processor.ProcessTrade();
        }
    }
```

# What's the point?

You might wonder what use there is in going through all this trouble to end up with the same result as with the first class.
In fact, we have just inverted the dependencies of the TradeDataProcessor class. This means the `TradeDataProcessorDI` class can take in its constructor not only the `DataProvider`, `DataParser` and `DataStorage` classes, but also any class that implements the `IDataProvider`, `IDataParser` and `IDataStorage` interfaces.

```cs
    public class XMLDataStorage : IDataStorage
    {
        public void Persist(List<UserData> users)
        {
            using (XmlWriter writer = XmlWriter.Create("users.xml"))
            {
                writer.WriteStartElement("UserData");
                foreach(UserData user in users)
                {
                    writer.WriteElementString("Id", user.ID.ToString());
                    writer.WriteElementString("Name", user.Name.ToString());
                    writer.WriteElementString("Phone", user.PhoneNumber.ToString());
                }
                writer.WriteEndElement();
                writer.Flush();
            }
        }
    }
```

```cs
    public class Program
    {
        static void Main(string[] args)
        {
            var data = new DataProvider("data/rawData.txt");
            var parser = new DataParser();
            var storage = new XMLDataStorage(); // New data storage class

            var processor = new TradeDataProcessorDI(data, parser, storage);
            processor.ProcessTrade();
        }
    }
```
