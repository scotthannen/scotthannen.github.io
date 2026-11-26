---
layout: post
title:  "Testing Data Access and Stored Procedures"
date:   2016-02-19 07:21:58 -0400
category: blog
---

This is more integration testing than unit testing, but the idea is the same. I ​have a class that reads/writes using stored procedures, and I want to write tests for it. It’s much easier to debug this code in isolation. It saves time.

This assumes that

*   the stored procedures just provide access to data - any significant logic is in the application.
*   the class being tested has a single responsibility - providing access to the data.
*   The process for testing is the same as with any other tests - Arrange, Act, Assert.

So if I’m testing a method that inserts data into table, the steps would be:

*   **Arrange**
    *   Make sure there’s no data in the table that will interfere with what I’m about to test
    *   Prepare my input
*   **Act**  
    Call the method to insert my data
*   **Assert**  
    Verify that the data is in the table just as I expect

## A Few Tools to Help Write the Tests

Add an app.config to the test project, and add your connection string.

``` xml
<connectionStrings>
  <add name="DinosaurDb" connectionString="server=USCLWSQLXYZ;database=APP_DINOSAURS;Integrated Security=SSPI" 
       providerName="System.Data.SqlClient" />
</connectionStrings>
```

I’ll need a convenient way to execute SQL statements so I can delete data from previous tests and see if what I’ve inserted is there. So I put a static class in the test project:

``` csharp
    static class Sql
    {
        public static void ExecuteSql(string connectionName, string sql)
        {
            using (var connection = new SqlConnection(GetConnectionString(connectionName)))
            {
                using (var command = new SqlCommand(sql, connection))
                {
                    connection.Open();
                    command.ExecuteNonQuery();
                }
            }
        }

        public static T ExecuteScalar<T>(string connectionName, string sql)
        {
            using (var connection = new SqlConnection(GetConnectionString(connectionName)))
            {
                using (var command = new SqlCommand(sql, connection))
                {
                    connection.Open();
                    return (T)command.ExecuteScalar();
                }
            }
        }

        public static string GetConnectionString(string connectionName)
        {
            return ConfigurationManager.ConnectionStrings[connectionName].ConnectionString;
        }
    }
```

Set up before and clean up after

I’m testing a SQL repository class that inserts records into a table called `DINOSAUR` and returns the `DINOSAUR_ID` for the new record.

In order to keep my unit test data separate from other data, I’m only going to insert dinosaurs with names that begin with “unittest.” So before each and every test I need to make sure I delete any records where the dinosuar name begins with “unittest.”

To do that I’ll add a method marked with the `[TestInitialize]` attribute. That method will run before every test and will clean up any test data from my previous tests. I don’t want to leave unit test data behind when I’m done either, so I’ll add another method with a `[TestCleanup]` attribute and delete all of my test data there too.

This is part of ‘arranging.’

``` csharp
    [TestClass]
    public class DinosaurSqlRepositoryTests
    {
        private const string TestPrefix = "unittest_";
        [TestInitialize]
        public void Setup()
        {
            CleanupTestData();
        }
        [TestCleanup]
        public void Cleanup()
        {
            CleanupTestData();
        }
        private void CleanupTestData()
        {
            Sql.ExecuteSql("DinosaurDb",string.Format("delete DINOSAUR where NAME like '{0}%'",TestPrefix));
        }
    }
```

## The Tests

Now I can write some actual tests. First I want to make sure that when the repository inserts a record and returns an ID that the record really has been inserted with my data and that ID. This test will insert a dinosaur with a random name using the `DinosaurSqlRepository` and then query the database for the expected record:

``` csharp
    [TestMethod]
    public void DinosaurSqlRepository_DinosaurIsInsertedCorrectly()
    {
        var newDinosaur = new Dinosaur
        {
            Name = TestPrefix + Guid.NewGuid().ToString().Substring(0, 10),
            Period = "Triassic"
        };

        var repository = new DinosaurSqlRepository(Sql.GetConnectionString("DinosaurDb"));
        var id = repository.InsertDinosaur(newDinosaur);
        var matchingRecords = Sql.ExecuteScalar<int>("DinosaurDb", string.Format(
            "select count(*) from DINOSAUR where ID={0} and NAME='{1}' and PERIOD='{2}'",
            id, newDinosaur.Name, newDinosaur.Period));
        Assert.AreEqual(1,matchingRecords);
    }
```

If I insert an identical dinosaur I expect the procedure to return an ID for the existing dinosaur instead of throwing an error. So I can test for that by executing the same insert twice and making sure I get the same ID both times. This isn’t even a behavior of the `DinosaurSqlRepository`. I’m testing the stored procedure.

(It looks like a lot more typing but most is copied and pasted from the first one.)

``` csharp
    [TestMethod]
    public void DinosaurSqlRepository_DuplicateInsertReturnsSameId()
    {
        var newDinosaur= new Dinosaur
        {
            Name = TestPrefix + Guid.NewGuid().ToString().Substring(0, 10),
            Period = "Triassic"
        };
        var repository = new DinosaurSqlRepository(Sql.GetConnectionString("DinosaurDb"));
        var firstId = repository.InsertDinosaur(newDinosaur);
        var secondId = repository.InsertDinosaur(newDinosaur);
        Assert.AreEqual(firstId, secondId);
    }
```

It’s important that each test run in isolation without affecting other tests. That’s accomplished by the setup and cleanup methods that run before every test.

If the class executes a query to retrieve data then I can write a test that inserts specific records and then make sure that my class returns the data I expect mapped correctly to my model. We can also send a bad input and test that the procedure throws an error with the expected method.

I’ve done this whenever I’ve written data access classes over the past few years and it really saves time. When I make mistakes I find them right away. I’ve only written the code a few minutes earlier so it’s fresh in my mind. Once these classes are tested then they just work and I can move on to the classes that depend on them.
