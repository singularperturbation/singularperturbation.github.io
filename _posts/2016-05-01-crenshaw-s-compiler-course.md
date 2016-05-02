---
layout: post
title: Crenshaw's compiler course - Cradle (part 1)
tags:
- Nim
- Crenshaw
- compilers
---

I've been wanting to learn more about how lexing, parsing, and computer languages
work, and have been looking for a self-directed course.  I considered looking at
something at Coursera, read through a little of Ruslan Spivak's [Let's Build a Simple Interpreter](https://ruslanspivak.com/lsbasi-part1/)
series, and even thought about picking up a copy of the the venerable
[Dragon Book](http://www.amazon.com/Compilers-Principles-Techniques-Tools-2nd/dp/0321486811),
but decided to do something different:

- When I last looked at the LBASI series, it was still very early.  It's since filled
  out quite a bit with useful information, but is not (as of writing) complete.
- Coursera classes require too much time, especially if it's a course that has a
  fixed beginning / ending date.
- The Dragon Book is a classic, but it's intimidating (and pricey!).  For someone
  who wants an introduction (and coming from a non-CS background), this is probably
  not a good place to start.

I ultimately came across a series of posts by [Jack Crenshaw](http://compilers.iecc.com/crenshaw/)
about building a compiler from scratch, and I decided to give it a try.  Though
now several years old, {%sidenote side1 And written in Turbo Pascal, targeting
Motorola 68K assembly! %} I like the writing enough to give it a go.  So far, I've
only done Chapter 1, but am making good progress. I think that it will work well
for learning the basics of compilers.


Rules of the road for this adventure:

1) Target [LLVM IR](http://llvm.org/docs/tutorial/LangImpl3.html) instead of
   assembly.

2) Use [Nim](http://nim-lang.org) as the implementation language.  It's a pleasant
   language to use, not too dissimilar to Turbo Pascal to prohibit porting (i.e., not
   hardcore functional), and it has a nice [wrapper to the LLVM bindings](https://github.com/FedeOmoto/llvm)
   that I can use.

3) Re-implement Crenshaw's work as closely as possible, only differing where necessity
   or style dictates.  Don't use [any](http://nim-lang.org/docs/pegs.html) of the
   [libraries](http://nim-lang.org/docs/parseutils.html) [available](http://nim-lang.org/docs/lexbase.html)
   in the standard library (or outside) that would help with parsing / lexing, since
   I want the experience of building it myself.

I chose LLVM IR over an M68K emulator (or using x86 assembly) since I wanted to learn
more about LLVM, be able to run my compiled programs on a 'real' computer, and I
thought it would be less complex than learning full-blown x86 assembly.  We'll see
if this assumption actually holds. :) Now that that's said- [Chapter 1](http://compilers.iecc.com/crenshaw/tutor1.txt)
of Crenshaw's tutorial deals with what he calls the 'cradle' - certain routines
that he'll use over and over later in the course.

I've implemented this in the `cradle` branch of my Github repo for this project
[found here](https://github.com/singularperturbation/crenshaw/blob/cradle/src/cradle/common.nim).
There's not much to say about it (since it's all fairly basic routines for reading
in input character-by-character), but I can give an example of how I was able to keep the
new implementation close to the original:

### Crenshaw's implementation
{% highlight pascal %}
{--------------------------------------------------------------}
{ Match a Specific Input Character }

procedure Match(x: char);
begin
   if Look = x then GetChar
   else Expected('''' + x + '''');
end;
{--------------------------------------------------------------}
{ Recognize an Alpha Character }

function IsAlpha(c: char): boolean;
begin
   IsAlpha := upcase(c) in ['A'..'Z'];
end;
{--------------------------------------------------------------}
{ Recognize a Decimal Digit }

function IsDigit(c: char): boolean;
begin
   IsDigit := c in ['0'..'9'];
end;
{% endhighlight %}

### My implementation
{% highlight nim %}
proc Match*(x: char) =
  if LookAhead == x: LookAhead.GetChar
  else: Expected "'$#'".format($x)

proc IsAlpha*(c: char): bool = c.toUpper in {'A' .. 'Z'}
proc IsDigit*(c: char): bool = c in {'0' .. '9'}
{% endhighlight%}

While the Pascal implementation is a bit more verbose (because of the begin / end
blocks, as well as my sneakily using `strutils.format`), you can see that overall
very little has changed (as intended).  I can even use Nim's set functionality
to keep the `IsAlpha` and `IsDigit` functions largely the same.

On to part 2 (expression parsing)!
