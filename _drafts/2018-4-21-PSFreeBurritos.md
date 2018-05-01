---
layout: post
title: PSFreeBurritos
featured-img: burrito
---
# An Ode to the Perfect Food

I have heard that there are places where burritos are not common.  I can only imagine those are dark, bleak places full of sadness.  Here in south-east Texas at least, burritos are a food group.  Actually all of the food groups.  Such a delicious orchestration of beans and meat and cheese.  Smothered with red sauce?  Sure go to town!  Green chile sauce?  New Mexico says hi!  Just more cheese?  heh, you do you cowboy!  It's pretty hard to [screw up a burrito](https://medium.com/@luckyshirt/dear-guy-who-just-made-my-burrito-fd08c0babb57).  I once had one at an Irish restaurant (like I said, Texas).  The tortilla was made of potato somehow and it was full of corned beef.  But you know what?  It was still a delicious burrito!

# What does this have to do with PowerShell

I must admit, I kinda want to write a whole article about burritos now.  But this is a PowerShell blog so let's get down to business!  This topic was going to be a lightning demo at the 2018 PowerShell and Devops Global Summit, but we had just way too much awesome content.  A blog post is probably better anyway, you never want to rush a good burrito story.

## Get to the point already!

Ok, ok, here's the deal.  This is a post about using PowerShell to automate the submission of a web form.  Specifically the customer survey at [www.chipotlefeedback.com](https://www.chipotlefeedback.com).  The survey is a multi-page web form with all sorts of unique settings.

We're going to start with fiddler to figure out what all the traffic is supposed to look like, then move on to PowerShell and play with Invoke-WebRequest to submit the forms and handle the return objects.  And finally we'll look at building our own version of a survey all in PowerShell.  So let's get started! Oh and the finished product is, of course, [over on Github](https://github.com/murrahjm/PSFreeBurritos)

# Reconnaissance

The first thing we want to do is figure out what the web traffic looks like behind the scenes when we fill out the survey the normal way.  [Fiddler](https://www.telerik.com/fiddler) is a great tool for just this thing.  You start it up, make sure it's capturing traffic (it usually does this by default), then navigate your website like you normally would in your [browser of choice](https://lynx.browser.org/).  It will capture all the communication for later review.  In our case, we can capture all the back and forth communication for our web survey.  After filtering out all the rest of the noise we see something like this:

![Fiddler Capture](/assets/img/posts/PSFreeBurritos/fiddler.gif)

The layout shows the communication flow on the left side, in chronological order.  On the right side, you see the request information on the top, and the response data on the bottom for the selected data.  There are several tabs available to show you different parts of the request or response.  Since we're primarily interested in the form submission, the WebForms tab works well on the top half.  For the response, I like to use the WebView tab, as that will show me more or less what the response web page looks like.  That makes it easy to follow along as you try to correlate between fiddler and your web browser.

So looking at our fiddler trace, it seems relatively straight forward.  We appear to be submitting form data to the same URL.  We should be able to capture all that data from each request sent, and resubmit them in the same order.  Now we just need a nice orderly way to save all that data.  Enter everyone's favorite data structure format, XML!  HAHA just kidding, we're going to use JSON.  The below is an abbreviated list of objects, each object being the complete payload for a form submission.  So the first object is everything we need to submit the first form, the second for the second, etc.

```JSON
[   
    {
        "lang" : "en",
        "stay_main-pager":  "0",
        "nodeId":  "survey1",
        "ballotVer":  "2",
        "hmac":  null,
        "is_embedded":  "false",
        "defPgrAction":  "next",
        "onf_q_chipotle_survey_invitation_method_alt":  "10",
        "spl_q_chipotle_receipt_code_txt":  null,
        "forward_main-pager":  "Begin Survey"
    },
    ...
    {
        "stay_main-pager" : "14",
        "nodeId" : "survey1",
        "ballotVer" : "10",
        "hmac" : "",
        "is_embedded" : "false",
        "defPgrAction" : "next",
        "spl_q_chipotle_customer_first_name_txt" : "Jeremy",
        "spl_q_chipotle_customer_last_name_txt" : "Murrah",
        "spl_q_chipotle_customer_email_address_txt" : "murrahjm@gmail.com",
        "forward_main-pager" : "Finish"
    }
]
```

That's not too bad, we can just loop through that data and submit each payload in order right?  Well almost.  We're making the assumption that all the data we captured is static.  We know our survey answers are probably going to be static, but is there other stuff in there that might change?  Well to figure that out we'll just need to do a few more captures.  Which means a few more survey submissions, which means a few more receipts.  So if you're following along at home that means eating more burritos in the name of research!

What we find after we are done ~~stuffing our faces~~ with our research is that the `nodeId` and `ballotVer` fields seem to change.  In fact it looks like `nodeID` is unique for the whole session and `ballotVer` changes with each new form request.  So that's something we'll have to handle as we go along.

 What else do we need?  well there's a cookie in that first packet, so we'll want to make sure to grab that.

![MeLoveCookies](/assets/img/posts/PSFreeBurritos/fiddlercookies.png)

And that should do it!  Now let's write some code!

# Invoke-WebRequest

Before we start our little loop of form submissions, we need to make that initial connection.  This will give us our session cookies, our session URL, and our initial form page.  We'll start off with the second best cmdlet in Powershell, [Invoke-WebRequest](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-5.1)!

```Powershell
$url = 'https://www.chipotlefeedback.com'
#initial connection and retreive first formdata
#also save cookie info in the session variable
$webrequest = invoke-webrequest $url -SessionVariable websession
#form data is here:
$webform = $webrequest.forms[0]
$webform | format-list

Id     : surveyform
Method : post
Action : /?feedless-chipotle-rcpt-survey-61c7f7b3df122d691ee3e0ea094ee51d
Fields : {[stay_main-pager, 0], [nodeId, survey5], [ballotVer, 2], [hmac, ]...}
```

The two things we care about up there are the *Action* and the *Fields*.  The *Action* property is what we'll append to our initial URL to make sure we go to the same form every time.  The *Fields* property is of course our form data, so let's look at that one.

```PowerShell
$webform.fields

Key                                              Value
---                                              -----
stay_main-pager                                  0
nodeId                                           survey5
ballotVer                                        2
hmac
is_embedded                                      false
defPgrAction                                     next
i_onf_q_chipotle_survey_invitation_method_alt_10 10
spl_q_chipotle_receipt_code_txt
i_onf_q_chipotle_survey_invitation_method_alt_20 20
beginButton                                      Begin Survey
```
Ok that's a lot of stuff, but what we care about here is the `spl_q_chipotle_receipt_code_txt`, `nodeID` and `ballotVer`  Most of the other stuff we'll just keep in there.  Otherwise it all looks pretty similar to what we saved from our fiddler capture into our JSON file.  Oh, so interesting thing with `i_onf_q_chipotle_survey_invitation_method_alt_10` and `i_onf_q_chipotle_survey_invitation_method_alt_20`, those are actually the radio buttons.  To *select* a particular button we just include the root key name (the name above with the _10 or _20 part) and then use the value for the button we want (10 or 20).  We'll see that again in later pages as well.  No idea if that's a common way to do radio buttons, but it's pretty consistent here.  Not a huge complexity, but it helps to eat a few more burritos to have plenty of sample data.  Well, *help* is a strong word but it couldn't possibly hurt.

Where was I?  Oh right, `nodeID` and `ballotVer`.  So `nodeID` appears to be a session-like thing.  It sometimes differs between surveys but doesn't appear to differ between individual form submissions.  `ballotVer` is a little different.  At first glance it just looks like a page number or something, incrementing each time.  But upon closer inspection it appears to skip a value every now and again.  That's weird, so we'll just grab it from the return object and apply that value to the next form data.

So armed with all that good info we're ready to start submitting form data.

```Powershell
#get these initial values
$nodeID = $webform.fields.nodeId
$nextBallotVer = $webform.fields.ballotVer
#$formdata is our imported json data, converted to a PSObject because PowerShell is like that
foreach ($item in $formdata){
    #update nodeID field
    $item.nodeId = $nodeID
    #set current ballotVer to the value specified by the previous return
    $item.ballotVer = $nextBallotVer
    #PSObjects are cool but we need a hash table for Invoke-WebRequest
    #so we use some function I found on the web, IDK
    $hashtable = $item | ConvertPSObjectToHashtable
    #submit form data
    $return = invoke-webrequest -uri "$url$($WebForm.action)" -Method POST -Body $hashtable -WebSession $websession
    #write the returned ballotVer value to retrieve the next time around
    $nextBallotVer = $return.forms[0].fields.ballotVer
}
```
Did that work?  We didn't get any errors, so... yes?  Maybe?  Well follow along dear reader, and let's take a journey through our data to answer that question with a little more certainty!

# Finding Errors and Debugging

The funny thing about computers is that they tend to answer the questions that we ask, not necessarily what we meant to ask.  For instance, if you submit form data with Invoke-WebRequest, and look for a return code, what you are actually asking is "did the POST operation succeed?" to which the web server will likely return "yes, return code 200 = success".  However, what we might have really wanted to know was something like "was my data correct and accepted by the form submission process?"  That's something a little different.  The answer is in there, but in our test case it's buried a little deeper than a return code.  But first let's think about things that could go wrong with our data submission.

* The receipt code is invalid
* The receipt code has already been used
* The form data was not expected or sent out of order

The question then is, what does the data look like for each of these errors?  Well the best way to find that out is to do it!  Submit a bogus receipt code, submit a good receipt code twice, munge up your json doc and send that.  The error text will be somewhere in the return data from the Invoke-WebRequest call.  The tricky part will be finding it, but one thing at a time.

A really great way to look at something like this while our script is running is with a debugger.  We can have it pause right when we get our return payload and browse around our data while it's still in memory.  So let's do that!  [This article](https://blogs.technet.microsoft.com/heyscriptingguy/2017/02/06/debugging-powershell-script-in-visual-studio-code-part-1/) on debugging is a great primer to get started.  Also this [Pluralsight course](https://www.pluralsight.com/courses/debugging-powershell-vs-code) is pretty great also.  So go check those out if you're not entirely familiar with the VSCode debugger and come on back.  (Hint: that article takes about 1 burrito worth of time to read, so, you know.  Research.)

Ok, so now that you know ~~kung fu~~ debugging, let's get to it!

We'll test our invalid receipt code scenario first.  Let's do three things first:

* Set a break point right below this line in our main function:
```powershell
$return = invoke-webrequest -uri "$url$($WebForm.action)" -Method POST -Body $hashtable -WebSession $websession
```
* open up a new powershell script file to help launch our debugger:
```powershell
. .\get-freeburritos.ps1
get-FreeBurritos -ReceiptCode 00000000000000000000 -answerfile .\sampleformdata.json
```
* Open fiddler and set it to start capturing so we can gather the web data as we hit our break point

![debugger breakpoint](/assets/img/posts/PSFreeBurritos/debug1.png)

Once we hit our break point, we should have one captured packet in fiddler.  Let's grab the raw output of the return packet and copy/paste it over to notepad to do a quick search.

![fiddler packet raw tab](/assets/img/posts/PSFreeBurritos/debug2.png)

There's a ton of data there but let's see if we can get lucky.  Let's search for the word "error" and see what we find.

![error found in notepad](/assets/img/posts/PSFreeBurritos/debug3.png)

Wow we found something!  `error-block readable-text` that sounds like an error message.  Ok now that we have that element, let's go over to our debugger and see if we can find that section.  We have the $return variable that should be populated with all this data, so we can just browse that like so:

![navigating return variable](/assets/img/posts/PSFreeBurritos/debug4.gif)

Look at all that data!  That message is really buried under there.  But that's ok, we've found it so we know what to look for now.  Cleaning up our request we can do something like this now in our script:

```powershell
$ReturnMessages = $return.allelements.where{$_.tagName -eq "BODY"}.innerHTML
if ($ReturnMessages.where{$_ -like "*class=`"error-block readable-text`"*"}){
    Write-error $ReturnMessages.split('<').where{$_ -like "*class=`"error-block readable-text`"*"}.split('<>')[1]
    return
```

We're making the assumption here that the generic-sounding `error-block readable-text` class is in fact a generic error message.  Based on that assumption we'll just output the error message as a powershell error object, then bail out of our script. (It turns out this covers scenario 3 as well, so that's cool)

What's next?  Ah, what about a re-used receipt code?  Well the same process applies more or less.  We'll do our debugger and our fiddler trace, then look at our return packet.  This time let's look at the web view.

![web view with receipt code error](/assets/img/posts/PSFreeBurritos/debug5.png)

That looks like some error text to me.  I'd bet that's probably in the same place as the other message, so let's go straight to our debugger and query that returnmessages variable.

```PowerShell
$ReturnMessages = $return.allelements.where{$_.tagName -eq "BODY"}.innerHTML
$ReturnMessages.where{$_ -like "*receipt code you entered has already been used*"}
```

![debug receipt code error](/assets/img/posts/PSFreeBurritos/debug6.gif)

Wow that's a lot of output.  That would take some work to filter down to just that message, but fortunately we don't have to do that.  Just the existence of that text is enough to tell us that we had an error, so we can just craft our own error message and call it a day.

```Powershell
write-error "The Receipt code you entered has already been used to complete a survey"
return
```

## Enough errors, what about success?

Debugging is like, such a drag man, so much negative thinking with, like, the errors and stuff.  Got to think positive man, good vibes and such.

Knowing when we hit an error is great, but we probably want to know when we succeed too.  In theory if we don't get an error we are probably completing the form successfully, but it never hurts to double check.  If we do another fiddler trace of a successful submission with our debugger on, we see this at the end.

![fiddler web view thank you message](/assets/img/posts/PSFreeBurritos/debug7.png)

That sounds definitively successful to me, so let's see if we can find that in the debugger output
```powershell
$ReturnMessages = $return.allelements.where{$_.tagName -eq "BODY"}.innerHTML
$ReturnMessages | where-object{$_ -like "*Thank You*"}
```
![searching for the thank you message](/assets/img/posts/PSFreeBurritos/debug8.gif)

Bingo, there's our success message.  If you were following along at home, as you were stepping through the debugger you might have noticed another piece of data that blinked by.  Let's look again:

![return variable in call stack](/assets/img/posts/PSFreeBurritos/debug9.gif)

Did you catch that `surveyprogress` value?  That looks like an incrementing value between 0 and 100.  What could we do with that?  ooh how about a progress bar!

![progress bar](/assets/img/posts/PSFreeBurritos/debug10.gif)

Muahahahahahaa!! Gimme those free burritos baby!  Packaged all up now we've got ourselves a script.  Well, I've got a script.  But I want everyone to have free burritos.  Like a burrito fairy or something.  Up to this point we've been interacting with our saved JSON file.  It works, but it's not the most user friendly thing in the world.  Ideally anyone using this tool would be able to craft their own JSON file by answering survey questions, without the need for a fiddler trace and all that.  The next section will focus on all that.  But in the meantime all that debugging left me famished.  Time for a burrito snack!

# The Survey



# Profit!