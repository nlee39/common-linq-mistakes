# Common LINQ mistakes

This is a list of common [LINQ](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/) mistakes, which has grown out from reading code of others (and myself) - and using the results for training people new to .NET.

If you complain about some of these "compiler/interpreter will fix that", I have to repeat what I've said before: "Programmer should be smarter than his/her tools". Not to mention, in many cases the misuse obfuscates the intent of the code, or returns invalid results.


## Deferred execution

By far the biggest issue people have when learning LINQ.  

[There are great many explanations](https://www.google.com/search?q=linq+deferred+execution), for example (in no particular order):  
* [LINQ and Deferred Execution](https://blogs.msdn.microsoft.com/charlie/2007/12/10/linq-and-deferred-execution/)
* [Deferred query execution](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/ef/language-reference/query-execution#deferred-query-execution)
* [Deferred vs Immediate Query Execution in LINQ](http://www.dotnetcurry.com/linq/750/deferred-vs-immediate-query-execution-linq)
* [Deferred execution vs immediate execution in LINQ](https://dotnetvibes.com/2016/01/04/deferred-execution-vs-immediate-execution-in-linq/)
* [Understanding LINQ's Deferred Execution](http://www.codeguru.com/csharp/csharp/cs_linq/article.php/c16935/Understanding-LINQs-Deferred-Execution.htm)

## .Where().First()

**Wrong:**

```
var person = persons.Where(x => x.LastName == "Smith" && x.FirstName == "John").First();
```
**Correct:**
```
var person = persons.First(x => x.LastName == "Smith" && x.FirstName == "John");
```
**Explanation**  
Depending on optimizations of the LINQ provider, this may mean, for example, that your database fetches all John Smiths to your application, and the application then returns first of those. You definitely don't want that.

This also applies for `.Where(x => x.LastName == "Smith").Count()` and other similar constructs.
  
## Using .SingleOrDefault()/Single() instead .FirstOrDefault()/First()
**Wrong:**
```
var person = persons.SingleOrDefault(x => x.Id == 21034);
```
**Correct:**
```
var person = persons.FirstOrDefault(x => x.Id == 21034);
```

**Explanation**  
`.Single()/.SingleOrDefault()` ensures that there is just one and only one match in the set, whereas `.First()/FirstOrDefault()` returns the first match.   

`Single..` methods go over all elements of the set, validating that there is just one match and throw an error if there are more - which is both costly and usually unneeded, unless validating user input or such.  

 In terms of SQL, Single is: `SELECT TOP 2...` whereas First is `SELECT TOP 1...`. If you have 10M rows, and your match is the very first row, Single will still run through all of the rows, validating that there are no more matches, whereas First returns immediately.

In most cases, you aren't interested in ensuring that there is just one match - or often it is logically impossible. See the examples above - if your database has two Person entries with same ID, you have far bigger problems than using LINQ badly...


## Not understanding the difference between First() and FirstOrDefault()  

This is highly situation-dependent. Sometimes .First() is correct, and sometimes it isn't.

It is important to understand that .First() and .Single() throw an exception when no match is found, whereas .FirstOrDefault() and .SingleOrDefault() won't &ndash; and return default/null value, depending on the data type.

Take, for example:
```
var person = persons.First(x => x.Id == 21034);
```
Exception may be the correct behavior if person with ID 21034 should exist, and not finding the entry is literally exceptional. 

On the other hand:
```
var login = users.First(x => x.Username == "john.smith@example.com");
```
Instead of having First() throwing an exception, you probably should use .FirstOrDefault() and check `login` variable for null - and log the invalid log-in attempt along with the details.



## Using .Count() instead of .Any() and .All()
**Wrong:**
```
if (persons.Count() > 0)...
if (persons.Count(x => x.LastName == "Smith") > 0)...
if (persons.Count(x => x.LastName == "Smith") == persons.Count())...
```
**Correct:**
```
if (persons.Any())...
if (persons.Any(x => x.LastName == "Smith"))...
if (persons.All(x => x.LastName == "Smith"))...
```
**Explanation**  
.Count() will do the full iteration of the (matching) items in the set. If you are interested that there is a matching item in the set, use .Any() - .Any() just returns true when there is at least one matching item, with an empty .Any() just returning true when there are items in your set.  

E.g. if you have list of 10M persons, and you need to know that a person with last name Smith exists, `persons.Count(x => x.LastName == "Smith")` will go through all the persons and check their last name, returning total number of Smiths. `persons.Any(x => x.LastName == "Smith")` returns true after encountering the first entry with last name Smith.

.All() is the opposite of .Any() - if you need to know that all your persons have lastname Smith, `persons.All(x => x.LastName == "Smith")` will return false when encountering first person with last name **not** Smith, without going over all of the set.


## .Where().Where()
**Wrong:**
```
persons.Where(x => x.LastName == "Smith").Where(x => x.FirstName == "John");
```
**Correct:**
```
persons.Where(x => x.LastName == "Smith" && x.FirstName == "John");
```
**Explanation**  
Chaining .Where() entries may end up in SQL database as multiple embedded queries (in style of `SELECT * FROM (SELECT * FROM PERSONS WHERE LastName = 'Smith') WHERE FirstName = 'John'`, instead of `SELECT * FROM PERSONS WHERE LastName = 'Smith' AND FirstName = 'John'`) &ndash; or will do multiple iterations over sets.  

It also obfuscates the intent of the query.


## .OrderBy().OrderBy()
**Wrong:**
```
persons.OrderBy(x => x.LastName).OrderBy(x => x.FirstName);
```
**Correct:**
```
persons.OrderBy(x => x.LastName).ThenBy(x => x.FirstName);
```
**Explanation**  
The "wrong" version sorts persons by LastName - and then ignores that sort completely and re-sorts persons by FirstName. Correct use is one .OrderBy() followed by .ThenBy(), which can be chained multiple times (`persons.OrderBy(x => x.LastName).ThenBy(x => x.FirstName).ThenBy(x => x.Age)`).  

This also applies to .OrderByDescending() and .ThenByDescending().


## .Select(x => x)

**Wrong:**
```
persons.Where(x => x.LastName == "Smith" && x.FirstName == "John").Select(x => x);
```
**Correct:**
```
persons.Where(x => x.LastName == "Smith" && x.FirstName == "John");
```
**Explanation**  
I am not sure why people just select the items themselves without any modifications &ndash; but I have seen this a couple of times. Perhaps they intended to do something with .Select() and forgot, or thought that .Select() materializes the query, like .ToList() or .ToArray().


## Empty .Count() for arrays, List&lt;T&gt;, Dictionary&lt;T&gt;, ...
**Wrong:**
```
var count = peopleArray.Count();
var count = peopleList.Count();
var count = peopleDictionary.Count();
```
**Correct:**
```
var count = peopleArray.Length;
var count = peopleList.Count;
var count = peopleDictionary.Count;
```
**Explanation**  
Don't use [Enumerable.Count()](https://msdn.microsoft.com/en-us/library/bb338038.aspx) method for arrays or any collections that implement ICollection&lt;T&gt;/ICollection interface, such as List&lt;T&gt; and Dictionary&lt;T&gt;. Use [.Length](https://msdn.microsoft.com/en-us/library/system.array.length(v=vs.110).aspx) for arrays and [.Count](https://msdn.microsoft.com/en-us/library/5s3kzhec(v=vs.110).aspx) property for ICollection implementations.  

Depending on platform, there are optimizations in place (see [here](https://referencesource.microsoft.com/#System.Core/System/Linq/Enumerable.cs,1191)) preventing O(n) operation of .Count() method, just returning existing .Count value.
