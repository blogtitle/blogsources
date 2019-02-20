---
title: "Rob's personal history of programming languages"
date: 2019-02-06T19:39:41+01:00
categories: ["Software Engineering"]
tags: ["Software", "Programming Languages"]
authors: ["Rob"]
draft: true
---

My total experience with coding as of today is 14 years. The time spans in the titles of this post might overlap.

# Pascal (2 years)

My first language was Pascal, I was 13 and was using Turbo Pascal at school to learn how to code.

It wasn't long until I realized I really loved coding. After two years my programming teacher complimented me for writing my own sorting algorithm that could run a little bit faster than what we studied at school.
I created something very similar to [Shell Sort](https://en.wikipedia.org/wiki/Shellsort), and in class we used to sort things with Bubble Sort.

It's been 12 years and I'm still unable to determine the time complexity of my algorithm, but this is a story for another time.

If you've never seen Pascal, it roughly looks like this:
```pascal
while a <> b do  WriteLn('Waiting');

if a > b then WriteLn('Condition met')
           else WriteLn('Condition not met');

for i := 1 to 10 do
  WriteLn('Iteration: ', i);

repeat
  a := a + 1
until a = 10;

{this is a comment}

case i of
  0 : Write('zero');
  1 : Write('one');
  2 : Write('two');
  3,4,5,6,7,8,9,10: Write('?')
end;

if i in [0..3, 7, 9, 12..15] then WriteLn('Match!') end;
```

# VB.Net (5 years)
After Pascal I switched to VisualBasic.Net, which was much more satisfying because it allowed me to drag-n-drop some elements into a window and create GUIs with it. It also had high level constructs and control flows like classes, inheritance, exceptions, and events.

I used VB.Net for more than 5 years, and kind of still think it was beautiful:
```basic
Namespace Sorting
    Module Program
        Sub Main()
        ' Removed
        End Sub
    
        ''' <summary>
        ''' Sorts an array with bubble sort
        ''' </summary>
        Sub bubbleSort(ByVal dataset() As Integer, ByVal n As Integer)
            Dim i, j As Integer
            For i = 0 To n Step 1
                For j = n - 1 To i + 1 Step -1
                    If (dataset(j) < dataset(j - 1)) Then
                        Dim temp As Integer = dataset(j)
                        dataset(j) = dataset(j - 1)
                        dataset(j - 1) = temp
                    End If
                Next
            Next
        End Sub
    End Module
End Namespace
```

Important features that I still like very much:

* The _wordyness_ of this code: there are almost no symbols, it is just words
* Closing blocks has sugar: `End` is always followed by something and you don't spend time looking for a matching `{`
* There are **no semicolons**: to not end a statement you can use underscores
* Docs is in the code in a well-defined way
* You could ReDim arrays to grow or shrink them both preserving and not preserving values. This is similar to how Go slices work
* Smart format strings (no need to know the type of what you are printing)
* Types are on the right: declarations all start with the same word or use `:=`

Of course, after using VB.Net for a while and with the advent of Stackoverflow-driven-development, I realized that almost no one in the world made my same choice, and was using C# instead.

Since C# and VB.Net could interoperate, I gave C# a try, and found it **AWFUL**.

# C#/Java (5 years)

Words had been replaced by symbols, braces were everywhere, semicolumns were a pain in the a\*\* conditions in statements required round parentheses around them, every declaration+initialization required you to write the type twice... It was terrible.

This is what C# looks like:
```cs
namespace Sorting 
{
    class bubblesort 
    {
        static void Main(string[] args)
        {
        // Removed
        }
    
        public static void SortArray(int[] array) 
        {
            int length = array.Length;
            int temp = array[0];
            for (int i = 0; i < length; i++) 
            {
                for (int j = i+1; j < length; j++) 
                {
                    if (array[i] > array[j]) 
                    {
                        temp = array[i];
                        array[i] = array[j];
                        array[j] = temp;
                    }
                }
            }
        }
    }
}
```

When I saw this my first reaction was: "What ON EARTH is THAT"
```
                    }
                }
            }
        }
    }
}
```

And the second one was, of course: "Do I have to put `public static void` there?"

It puzzled me for a long time to see that C#, Java, C++, C and all languages in the family were so popular and yet so hard to read.

When I went to university, I had to study C and Java. C was not that problematic if you don't consider pointer arithmetics, the lack of strings and the fact that if you don't check your boundaries the kernel is going to kill you.

Java was basically like a featureless C#, so the pain did not end.

As much as I hated them I was able to complete some complex projects that did shady things with the network, some video games, and some tools. After all I was mostly satisfied with the overall experience.

The pain never really went away. If anything, it increased: multithreading in C# and Java was madness (it was the era of multi-cores on anything) and dealing with bare byte streams meant writing **everything** from scratch.

# Python (2 years)

That is what brought me to use python. Python for me was pure eye-candy: no semicolons, no noise, less symbols, libraries for everything I needed, with a big, great **BUT**. There were no types and way too much magic. I felt threatened by every line of code that I wrote. I was always puzzled by the meta programming surprises that I could encounter.

Multithreading was even worse than Java, performance made me cringe every time and relying on invisible characters for semantics is just silly.

```python
# BubbleSort!
def bubbleSort(alist):
    for passnum in range(len(alist)-1,0,-1):
        for i in range(passnum):
            if alist[i]>alist[i+1]:
                temp = alist[i]
                alist[i] = alist[i+1]
                alist[i+1] = temp
```

At that point I was just desperate. I talked with some friends about writing a new high-level language with multithreading and simplicity in mind, with no meta programming, no magic, but with strong types. I wanted to always know what was going on: no annotation, no overload, no going around looking for the actual implementation of a function (like I did for Java generics and interfaces and subclasses). 

I just wanted to focus on the code, have a type system to rely on, and never have any surprises.

That's when [Anna](https://blogtitle.github.io/about/#anna-https-blogtitle-github-io-authors-anna) introduced me to Go.

# Go (4 years and counting)
I have to be honest: the first impact with Go was not good. 

* The lack of exceptions was painful (took me a while to figure out [it is better to not have them](https://blogtitle.github.io/considerations-on-error-handling/)
* The lack of answers on Stackoverflow was astonishing.
* Syntax was still very C-like: no semicolons and no parentheses around conditions, but braces were still there.
* No constructors

I still decided to try it out, and after a long while, it clicked. It took me about two months to realize I was solving my problems, and I was not thinking about language-related stuff. I didn't have to care about irrelevant details and I could just focus on what I was solving.

Noise went away (`public static` is just an uppercase letter, and `void` is literally no characters), and the standard library is so vast and well documented I almost never needed to look for anything else.

This is what a bubble sort looks like in Go:
```go
func bubbleSort(data []int) {
	for range data {
		for j := 0; j < len(data)-1; j++ {
			if data[j] > data[j+1] {
				data[j], data[j+1] = data[j+1], data[j]
			}
		}
	}
}
```

If you want to know why I stayed with go, read [this](TODO).

After I met Go, I also got to know JavaScript, Typescript and Rust and use them for a while.

I find JavaScript and everything that revolves around it a sickness of modern software development, and I find Typescript a nice effort to make JavaScript more palatable.

I find Rust a nice tool to write low-level and real-time programs, but it is still too complicated and too low-level for most projects. Multithreading is almost not there and the standard library is a little bit too thin for my taste. Overall I like it, but I wouldn't use it as my first language for most projects. I will write something more specific about Rust in the future, but I feel like I still need to learn more to talk about it.
