---
layout: post
title: Hash Tables and Dictionaries
featured-img: PowerShell
categories: [PowerShell]
---

# When is a Hash Table not a Hash Table

When it's ~~ajar~~ a dictionary!  It would seem that today is data type week, stuff like this keeps coming up.  I was just reading about arrays vs lists and then ran into this issue with hashtables and dictionaries.  There must be something in the water.  Anyway, since this resulted in hours of confusion for a simple misunderstanding, and since I'm just as likely to make this same mistake 6 months from now, it seemed like the kind of thing I should write down.

## What is a Hash Table?

[about_hash_tables](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_hash_tables?view=powershell-6) defines a hash table as "a compact data structure that stores one or more key/value pairs" and says "...also known as a dictionary..."  Basically it looks like this:

```PowerShell
@{one=1;two=2}

Name                           Value
----                           -----
one                            1
two                            2

```
You can reference the parts of a hash table like this:

```PowerShell
$hashtable.keys
one
two

$hashtable.values
1
2
```

And you can add and remove items with the handy 'add' and 'remove' methods

```powershell
 $hashtable.add

OverloadDefinitions
-------------------
void Add(System.Object key, System.Object value)
void IDictionary.Add(System.Object key, System.Object value)

$hashtable.remove

OverloadDefinitions
-------------------
void Remove(System.Object key)
void IDictionary.Remove(System.Object key)
```

If you look at get-member you see we clearly have a hashtable

```powershell
$hashtable | gm

   TypeName: System.Collections.Hashtable
```

## What is a Dictionary?

So that all seems pretty straight forward.  But then, what is a dictionary?  Well that depends.  Let's look at one example, say `$PSBoundParameters`.  If you're not familiar, `$PSBoundParameters` is an automatic variable that is populated inside a running script or function, and contains the names and values of any specified parameters.  It's a great way to determine if a user provided a value to a parameter at run time.  It only exists inside the running script or function, so it's tricky to catch, but if we use a debugger or if we just output the contents, we can see something like this:

```PowerShell
Function Test-Hashtable {
    Param(
        $one,
        $two
    )
    $PSBoundParameters
}
Test-hashtable -one 1 -two 2

Key Value
--- -----
one     1
two     2

```
Hey that looks familiar doesn't it? Let's grab that into a variable and play with it.

```PowerShell
$output = Test-hashtable -one 1 -two 2

$output.keys
one
two

$output.values
1
2

$output | gm

   TypeName: System.Management.Automation.PSBoundParametersDictionary

...
```

Yep, that all looks like what we'd expect, nothing funny going on.  That is totally a hashtab... wait what is that type?!  PSBoundParametersDictionary?  Weird, well a hash table is a dictionary right, so maybe a dictionary is like a hash table?  Let's check a few of those methods we used with our hash table, maybe it's more or less the same thing

```PowerShell
$output.add

OverloadDefinitions
-------------------
void Add(string key, System.Object value)
void IDictionary[string,Object].Add(string key, System.Object value)
void ICollection[KeyValuePair[string,Object]].Add(System.Collections.Generic.KeyValuePair[string,System.Object] item)
void IDictionary.Add(System.Object key, System.Object value)

$output.remove

OverloadDefinitions
-------------------
bool Remove(string key)
bool IDictionary[string,Object].Remove(string key)
bool ICollection[KeyValuePair[string,Object]].Remove(System.Collections.Generic.KeyValuePair[string,System.Object] item)
void IDictionary.Remove(System.Object key)
```

## Close enough, what's the big deal?

It's got a little more going on in the way of overloads, but the basic methods are the same right?  Ok raise your hand if you saw the problem the first time.  Now raise your hand if you spent hours staring at your code before you noticed the difference.  **raises hand**

the PSBoundParametersDictionary object returns a boolean value (presumably success/failure status) whereas the hashtable returns nothing. `[void]`. So when you have that line buried in a bunch of other code and you're trying to control your output you can't figure out why you keep seeing:

```PowerShell
True

Key Value
--- -----
two     2
```

__Note__:
> $PSBoundParameters is not the only dictionary type, there are plenty of others.  You can test this out with a generic dictionary type by using:
>
> ```$var = New-Object 'System.Collections.Generic.Dictionary[String,String]'```

# Conclusion

One of the strengths of PowerShell is that it makes it easy to do a lot of great things without having to really know much about data types.  Most of the time they don't matter.  But some of the time they do, and being aware of them can make your code better and potentially save you some heart ache.
