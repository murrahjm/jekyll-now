---
layout: post
title: review Ansible
featured-img: ansible
comments: true
categories: [Ansible]
---

Being raised in the "good 'ol days" (which let's be honest is anything before 2020 at this point), I grew up with that over-exaggerated humility that says you can't ever admit to being good at anything.
So when [Josh Duffney](https://twitter.com/JoshDuffney) asked me to review his new book, [become Ansible](https://becomeansible.com), I was surprised at first.
But I do know a [thing or two](https://youtu.be/ZI20Y10OKd0) about Ansible, and I'm pretty opinionated, so I figured this would be right up my alley.
So, for better or worse, here is my review.

## This is not a book about Ansible

Being a fan of Ansible I jumped right in to this book, expecting a good time.
Over the first few chapters I was struck with two competing thoughts.

1. Whoa, this is super dense, I better sit up and pay attention.
2. Where's all the Ansible?

To answer that second one I went back to the beginning of the book that I unconsciously skipped.
Reading the what and the why of the book, then flipping through the table of contents, it started to make sense.

Like so many tech books I've read over the years, I was expecting an in-depth treatise on a single technology.
I expected it to start with the basic elements, then quickly expand into all the corners, teaching me everything, or nearly everything, there is to Ansible.
But this is not a book entitled *everything about Ansible*, it is a book entitled *become Ansible*.
To *become Ansible* is to become part of something larger.
SRE, devops, whatever you want to call it, it is to derive value from the combination of many tools and processes.
Ansible itself is not a tool meant to be mastered in a bubble.
It's very existence supposes an entire infrastructure to manage.
Mastery of that infrastructure, in all its forms, is as vital as mastery of any one tool.
To *become Ansible* is to be a sysadmin with an array of technologies at your disposal.
What can you build with an array of tools?
Anything. Everything.

So with that realization (and a smile), I started over.

## Containers and Clouds

As un-cool as it is to admit this in 2020, I know very little about containers.
They've always been just a step away on my future list of *things to learn*.
So I was definitely looking forward to learning a thing or two in this first section.

The book layout reminds me of textbooks, or workbooks, with a very clear list of objectives at the start of each chapter, and a review section at the end.
It's also designed for you to follow along, with clear instructions on commands to run and why.
So I started following instructions and they just worked!
In no time I had a container running with all the pieces I needed to continue on.
Following along I made a more and more complex command line for running my new container, and even published it to docker hub.
It was a delight to walk through this, and it made for a very quick tutorial.

Continuing, it was time to do cloud stuff.
The book has two options, AWS and Azure.
I chose the latter and continued the lesson at that chapter.
Before I knew it I had azure resources built in the cloud, and felt very devops-y.
At this point the commands become more complex than one-liners printed in the book, and I started running the playbooks that were included in the code folders.
The code here is very well organized, separated into chapters, and named well to make it easy to find.
It was nice to be able to open these folders in VSCode and read through all the playbooks prior to, and while, running them.
> **Note:** I am very appreciative of the included playbooks for deleting the azure and aws environments.
My good looks only get me so much extra money, so not having to pay for a bunch of cloud resources to sit around is nice.
That didn't need to be included, but it is a nice touch, and very helpful.

## Playing with Code

Now that the development environment is all set up, we are ready to start doing some coding.
This section starts right off with an intro to git.
This is such a foundational technology, and I'm glad he included it.
I can't say truly from a beginner's perspective, but it feels like it strikes a good balance of explaining just enough to be effective, without getting down into the weeds.
Again, this is just another tool in the tool belt.
If you are new to git, definitely follow along and pay close attention to this chapter.

The rest of this section dives into core Ansible functionality.
I don't want to give away any of the goodies, but Josh does a great job of laying things out in a progressive manner.
I especially like the way it starts with ad-hoc commands for simplicity, then progresses into playbooks as those grow more complex.
This book also gives equal time to both linux and Windows, which mirrors what a sysadmin might expect to see in many environments today.
And, as it should, the next several chapters cover everything you'd expect from a book about ansible.
Playbooks, inventories, roles, secrets, etc.

## Playing with Pipelines

A chapter on CI/CD is such a great way to wrap up this book, and I think I was most excited about this one.
The chapter focuses on GitHub Actions, but these same methods could be applied to many other products.
And, true to form, it references all the previous work, and even comes full circle, back to the containers built at the beginning.
This is the toolchain, of which Ansible is one part.
The example used here is sufficiently complex to give a great overview, and I'm excited to do more things with GitHub Actions now.

In a way, I wanted there to be more of this.
I feel like this is an important enough concept that it deserves more than a single chapter.
However, it really does cover a lot of ground in that one chapter, and hits enough bullet points that anything else could be repetitive.

## Conclusion

A couple of things stood out to me with this book.
It is simultaneously complex and accessible.
This is no "hello world" example, and is going to require some critical thinking to really apply these lessons and follow along.
If you are blindly skimming and copy/pasting commands you're going to have some trouble.
But if you are singularly focused and intent on learning, this book can dump some great knowledge.

It is also surprisingly succinct.
No fluff about the author's random opinions on the weather, or a history of the Internet.
No historical lesson on the importance of bash and how the syntax works.
It feels like there is an assumption that we either know these basic elements (how to run commands, bash, powershell, etc.) or that we are smart and can figure it out on our own.
Either way, I appreciated the brevity.
I was also impressed that every error I ran into was expected and documented.
I'd get excited when something failed, thinking I could send 'ol Josh a list of things that need fixing.
So I'd write it down and make some notes, only to scroll down half a page and see the detailed explanation waiting patiently for me.
I also really liked the non-Ansible sections.
As I mentioned in the intro, those are just as important in modern IT shop as anything else, and this book was a fantastic overview.
I'm still a beginner with things like docker and GitHub Actions, but I feel like I have a much better handle on how to dig into those.

On the other hand, I can say this book feels like it works more in the workbook / lab type of environment.
This isn't something that you're going to read casually while drinking a cup of coffee in the morning.
That's not necessarily a downside, but if you're a weirdo like me who enjoys technical documentation and a danish, you might find it to be not a great fit for that.

In short, I am excited to have read this book, and I want to thank Josh for writing it.
The word that kept coming to mind is 'joy'.
It was a joy to travel through this book.
It was a joy to watch the containers and cloud infrastructure build out like magic.
It was a joy to discover different ways of organizing tasks in playbooks.
It was a joy to watch my CI/CD pipeline just do its thing.

Take a look at the table of contents. If you get excited at the prospect of geeking out on all this technology, then you may also find joy in this book.
It's the eleventieth month of 2020, and it feels like the world is falling down around us.
But I had fun following along with this book, and learned a few things while simultaneously escaping from reality for a bit.
It doesn't get much better than that.
