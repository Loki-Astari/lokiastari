---
title: Interviews Processes
author: Loki Astari, (C)2011
date: 2011-10-21T20:01:23-0800
comments: true
categories:
- Interviews
- Work
description: One of the things the most non-programmers are surprised about is the
  severity of programming interviews. So why are they so intense?
draft: false
cover:
  image: /images/post/post-1.png
  hidden: false
  caption: Photo by Possessed Photography
---

We ([Moz.com](https://Moz.com)) have been conducting many interviews with new engineers lately. I have been asked to help find great new team members among the hordes of applicants. Luckily, we have a great team that pre-screens applicants before they get to engineers (thanks, you guys).

One of the things most non-programmers are surprised about is the severity of programming interviews.

An on-site interview (after you have passed all the phone screens) consists of seeing between six and eight people (depending on the company); this will take a full day, and you will be utterly exhausted.
But why are "Software Engineer" interviews so exhaustive? [I think this sums it up](https://programmers.stackexchange.com/questions/47778/why-are-sw-engineering-interviews-disproportionately-difficult-vs-research-inte/47784#47784)

### What types of interviews do I do?
My field of expertise is technical (not the "[touchy feely](https://blog.sironaconsulting.com/sironasays/2011/03/is-your-hr-manager-more-miss-marple-or-hr-20-fun-infographic.html)" HR stuff or ["can you work as a team"](https://imgur.com/gallery/4D6wd) management stuff). As a result, I stick to one type of interview with two sub-variations:

* Can you think on your feet
 * Do you know the basics (algorithms/data structures/Big O/CS theory)
 * Design interacting systems at a high level, thinking about the interaction

I have not done phone screens in a while (this is a different skill set). I have no problem doing them; I think other people are better than me at this interview, as I find it hard to get technical information over the phone (even if you use one of those collaborative online writing tools). I would rather render my ~~judgement~~ opinion on the candidate based on talking to them face to face.

### So how do you get good information from a candidate?
Heck, if I know. During the interview, it is a lot of gut feeling and pressing the candidate to express themselves and provide feedback on what they are thinking. **Please**; <span style={{color: 0x993366}}>if you are a candidate explain what you are doing*</span>, I want to know how you got the solution more than I want to see the solution you provided.

When I ask a candidate about the basics, their inability to answer the question does not tell me much (you may not have used that technique in a while or missed that one class at college), so don't worry too much; I will find something you can answer. But note that your ability to answer the question only tells me a little more (not much).

So when you do answer I want to know **why** you are using a particular technique, are there **any other techniques** and if so **why** use your initial choice over another ('it would take too much wide board space' is an excellent answer as it means you understand the alternative is very complex (but now I am going to ask you to do it :-) ))

Unless the company is looking for a skill in a particular language, my interview questions generally are language agnostic. I will let the candidate use the language they are most comfortable with on a whiteboard. Though the language is not important to me, **<em style={{ textAlign: 'justify' }}>the usage of the language is important</em>** (you better not leak memory in C/C++, you better not build strings with java.lang.string in Java, you better know how to use regular expression matching in Perl, you should understand how to use the fundamental list/dictionary types in python etc.) and most importantly **you better be able to use fluently any language you list on your resume** (if you are learning/fiddling with a language make that clear).

Notice I have not said anything about syntax. Personally, I think this is irrelevant in an interview situation. I often have minor syntax errors as I code (that's why I have the compiler as my first line of defense and unit tests as the second). As long as I can understand what you are logically trying to achieve, I will not complain about missing a semicolon.

### The dos and don'ts for a candidate
This article has guidance on what to expect at [programming interview](https://programmers.stackexchange.com/questions/80065/preparing-for-interviews/80073#80073).

It mentions studying the company. When I was younger, I always thought this was silly (I was naive once a long time ago), as I did not think it would help me do my job. Still, I accepted my elders' advice and studied the companies before interviews, and I must say it always paid off (but never in the ways you expect).

Remember that I only have an hour to try and extract as much information as possible from you, so don't go off on wild tangents. Please stick to the point and answer the question I ask. If you can show off quickly, then do so, but make it quick. You should also note that interviewers generally have their questions arranged in themes, if you can answer the questions quickly he is going to move onto the next part of the same question which extends the question making it more challenging trying to draw more knowledge from you (can you make that more efficient/ can you see the common pitfalls how do you avoid them etc..).

On the same line don't give the most optimal solution first (unless that is obviously what the interviewer is looking for and you can justify it), a good interviewer is going to walk you up to the optimal solution and see if you know why it's the optimal solution (But it's also part of the communication I mentioned above, if you explain what you are doing (I will start with a brute force solution) even a bad interviewer should know you have an alternative optimizes solution).

Please don't gloss over the complex bit (unless the interviewer tells you too); this is probably the bit they want you to explain. Whatever you don't try, BS your answer. The interviewer probably asked the question a hundred times before; they have heard about all the good/standard ways of solving the problem (and some more exotic ways). If you don't know, just let the interviewer know.

### The don'ts for an interviewer
I hate interviewers who ask those silly logic problems. They do not tell you anything about a candidate's ability to write code or think critically about coding. All it tells you is that they are good logic problems.

* Why is a manhole cover round?
* If a man rows a boat at 4 mph downstream on a river traveling 2 mph. He drops his hat overboard but does not notice for an hour. How long does he need to row upstream to meet his hat?
* Russian roulette: I have a six-shooter with 2 bullets in consecutive locations. I spin the chamber, then shoot myself first (it does not go off). I pass the gun to you. Do you spin the chamber before shooting or not?
* You have 1000 bottles of champagne. One bottle is contaminated, and drinking from it will cause vomiting in an hour. You have an hour and ten staff to help you. What are the fewest bottles you need to discard to guarantee none of your guests gets poisoned?
* Many more.

I like the problems and do well at interviews that ask them. But I don't think it tells you anything about me as a software engineer. As a result, I don't ask these types of questions.

Other people argue that it shows people who can think critically about problems. I'm sorry, but I disagree. It tends to find people who have heard the problem before, and these problems appear on websites all the time.

### The coding interview
I previously did a second type. Sit the interviewee in front of a laptop and ask them to write code to solve a particular problem. This would involve user input (from keyboard or file (as user input was involved, I would look for error checking, but since a file was involved, no error recovery)). I would provide documentation to an existing library to see if they could read and understand it enough to use it. *But this is a blog entry for another time. But if you have some hints and comments about it, please post a comment below*.
