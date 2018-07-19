---
layout: post
title: Continue..Return..Break OhMy!
featured-img: Powershell
categories: [PowerShell]
---

Do you ever have one of those things that you know, until someone asks you about it?  Like when someone asks you to define [hegemony](https://www.vocabulary.com/dictionary/hegemony), or [pernicious](https://www.vocabulary.com/dictionary/pernicious).  Or in my case when I was talking to some coworkers about using `continue` and `break` in a script, and they asked me if I was sure that's how those worked.  Well it turned out I wasn't, not exactly.  I had a pretty good idea about how to use them, and hadn't run into any major issues, but I realized I couldn't explain the difference between `continue`, `break`, and `return`, not in any concrete way.  And you know what they say, the best way to learn something is to explain it.  So here, friends, is my humble attempt to explain to myself the specifics of the `continue`, `break`, and `return` keywords.

This is, of course, in no way a new topic.  In fact you can read the official documentation for each of these commands here:

* [continue](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_continue)
* [break](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_break)
* [return](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_return)

However, I wanted to get an idea of how they all worked together, in a compare/contrast sort of way.  And I haven't written any blogs in a while, so I wanted to get something written down anyway.  In exploring these I'll start with a simple example that doesn't offer much in the way of differences, then expand up to a larger script to illustrate where the divergence occurs.  So let's begin at the beginning.

## The Basic Script

For my examples I decided to stick with the `for` loop.  Things should work similarly in a `do/while` loop or a `foreach-object` loop.

```PowerShell
"**This uses the continue keyword**"
for ($i = 1; $i -lt 6; $i++)
{ 
    if ($i -eq 3){continue}
    $i
}

"**This uses the break keyword**"
for ($i = 1; $i -lt 6; $i++)
{ 
    if ($i -eq 3){break}
    $i
}

"**This uses the return keyword**"
for ($i = 1; $i -lt 6; $i++)
{ 
    if ($i -eq 3){return}
    $i
}
```
I have three loops here.  They are all identical except for the keyword used.  The output from above looks like this:

```
**This uses the continue keyword**
1
2
4
5
**This uses the break keyword**
1
2
**This uses the return keyword**
1
2
```

So right off the bat you can see how `continue` is different. `Continue` is telling the loop to stop what it's doing and go immediately to the beginning of the next iteration of the loop.  In this particular case, "don't go to the next line and output *3*, just go straight to the beginning of *4*".  `Break` and `return` appear to be doing the same thing, both are ending the entire loop.  Do not pass go, do not collect $200 as they say. (do kids still play Monopoly?) But why have two commands that do the same thing?

## Nested Loops

Not surprisingly it turns out they don't do the same thing, we just need a little bit more complexity to find it. In this case the best way to add complexity is to add more loops.

```PowerShell
    "**This uses the continue keyword**"
    for ($i = 1; $i -lt 6; $i++)
    {
    $i
        for ($j = 1; $j -lt 6; $j++)
        { 
            if ($j -eq 3){continue}
            "$i.$J"
        }
        
    }
    "**This uses the break keyword**"
    for ($i = 1; $i -lt 6; $i++)
    {
    $i
        for ($j = 1; $j -lt 6; $j++)
        { 
            if ($j -eq 3){break}
            "$i.$J"
        }   
    }
    "**This uses the return keyword**"
    for ($i = 1; $i -lt 6; $i++)
    {
    $i
        for ($j = 1; $j -lt 6; $j++)
        { 
            if ($j -eq 3){return}
            "$i.$J"
        }   
    }
```
Here I've nested each of the previous loops inside a parent loop.  The conditional test stays inside the inner loop to illustrate what happens to the output.
```
**This uses the continue keyword**
1
1.1
1.2
1.4
1.5
2
2.1
2.2
2.4
2.5
3
3.1
3.2
3.4
3.5
4
4.1
4.2
4.4
4.5
5
5.1
5.2
5.4
5.5
**This uses the break keyword**
1
1.1
1.2
2
2.1
2.2
3
3.1
3.2
4
4.1
4.2
5
5.1
5.2
**This uses the return keyword**
1
1.1
1.2
```
`Continue` is the same, it's skipping *3* and continuing with the rest of the loop.  Here, though, `return` and `break` are different!  What's going on?  Looking at `break` you can see that when it gets to *3* it is ending the inner loop entirely, but it still goes on with the outer loop.  So you'll never see *4* or *5* from our inner loop but you get all of the outer loop no problem.  This jives with what the documentation says:  "...the Break statement causes PowerShell to immediately exit the loop".

> In the [documentation](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_break) for the `break` keyword there's mention of using labels to specify which loop to break out of, rather than just the one loop the keyword is used in.  I hadn't ever heard of that either, or seen it used that I recall.  It's pretty interesting though, so go check it out.  For the purposes of this article I'll just stick with the default behavior

`Return` on the other hand just stops the whole thing.  "Danger Will Robinson" (I know the kids don't watch that anymore).  That script hits the first *3* and just bails all the way out. Or does it?  The `return` keyword can also be used to exit a function, so it can't just be bailing out of the whole script or that wouldn't work.  Back to our documentation, we see "The Return keyword exits a function, script, or script block".  (Ignore the whole "returning a value" thing for now, that's not really relevant here)  The `return` keyword is going all the way "out" or "up" to what it considers the top, which is either a function, script or a scriptblock.  So in this example `return` is, in fact, going all the way out and ending the entire script, because there is no function or scriptblock to stop it.

To illustrate this, I've added a function in between the two loops.

## Adding a Function

```PowerShell
function inner-loop ($i) {
    for ($j = 1; $j -lt 6; $j++)
    { 
        if ($j -eq 3){continue}
        "$i.$J"
    }
}

for ($i = 1; $i -lt 6; $i++)
{ 
    $i
    inner-loop $i
}
```

That looks a little weird so I'll break it down.  Previously the *i* loop was on the outside and the *j* loop was inside.  That's still the case, but now the *j* loop is inside a function, and the *i* loop is calling that function.  Running that with the `continue` keyword shows the full loop output minus the *3*s, just as we would expect.

```
1
1.1
1.2
1.4
1.5
2
2.1
2.2
2.4
2.5
3
3.1
3.2
3.4
3.5
4
4.1
4.2
4.4
4.5
5
5.1
5.2
5.4
5.5
```

Replacing the `continue` with `break` has more expected output:
```
1
1.1
1.2
2
2.1
2.2
3
3.1
3.2
4
4.1
4.2
5
5.1
5.2
```
`break` is breaking out of our inner loop just as you'd expect.  Now if I swap out the `break` with `return` will that still return the super short end-everything output?
```
1
1.1
1.2
2
2.1
2.2
3
3.1
3.2
4
4.1
4.2
5
5.1
5.2
```

Nope!  `Return` is going up to the top of its boundary, which, now, is the function.  The *i* loop is outside the function, so it is unaffected by the `return` keyword, and just keeps on happily looping.

For extra credit swap out the `return` keyword for `exit`.  That will cause all functions and all loops to end immediately.  Unfortunately, it will also close your PowerShell console so you may not actually get to see the results.

## fin

And that's about it.  So in summary:

* `continue` goes to the next iteration of the loop
* `break` ends all iterations of the current loop
* `return` goes all the way up and out of the function or script
* `exit` goes out of everything, including PowerShell

Not the most earth-shattering, ground-breaking, amazing new feature, but sometimes it's good to go back to the basics.  I learned a few things writing this, and I hope you learned a few things reading it.
