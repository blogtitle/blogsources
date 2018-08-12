---
title: "Splitting Pipelines"
date: 2018-07-13T21:08:41+02:00
categories: ["Scripting"]
tags: ["Bash"]
authors: ["Rob"]
---
# The problem

Some days ago I was looking at some configuration files that were presented in this form:

```ini
#
# This is a section
#
VALUE_ENABLED=y
# VALUE_DISABLED 
```

The problem in reading this stuff is that both comments and non-comments are relevant and none should be ignored since a disabled feature could cause as much trouble as an enabled one. 

I would have liked to see comments in a different file than the rest, so I thought: "Well, this is an easy job for grep".

# The approach
Since I hate to waste processes while I use bash, regardless the significance in term of performance, my first idea was to write something like this:


![information flow](/pipelines/process-sub-first.png)


The problem in this approach is that with tee it is not easy to just split a pipeline unless you know the `>(command)` trick (aka process substitution).

This is the bash version of the above graph:

```sh
#Take away empty lines and lines only containing white space
grep -Ev "^(|[[:space:]])$" configfile | 
#In a subshell Grep for lines beginning with '#'
tee >(grep -E "^#" >                    
	#Write result to file and resume the main shell 
	comments.conf) |                    
	#Grep for non-comment lines
	grep -Ev "^#" >                     
	#Write them to file
	non-comments.conf                   
```

In bash `>(command)` creates a file descriptor to which other commands __can write__ and `<(command)` takes the output of a command and creates a file descriptor from which other commands __can read__.

What is being written to the file descriptor is instead redirected to `stdin` of `command` as if it came from a pipeline. 

Since tee accepts files as parameters and not commands, this trick is needed if you want to avoid useless creation of FIFOs.

## Complication
This solution is mostly fine, it does what it is supposed to do, but doesn't do something that I realized later being useful: __keeping sections titles__ even in the non-comment file.

This:

```ini
#
# This is a section
#
VALUE_ENABLED=y
# VALUE_DISABLED 
```

Should've become this:

```ini
# This is a section
VALUE_ENABLED=y
```

Since all variables are composed only by upper-case text I changed the solution to be:

```sh
grep -Ev "^(|[[:space:]])$" configfile | 
tee >(grep -E "^#" > comments.conf) | 
grep -E "(^[^#]|^# [A-Z][a-z])" > non-comments.conf 
```

# Conclusion (TL;DR)
If you want to split a pipeline or generally write to the stdin of a command as if it was a file use `>(command)` i.e.

```sh
# all 3 commands receive the output of somecommand
somecommand | tee >(command1) >(command2) | command3
```

Even the reverse can be useful:

```sh
# Obtain differences between the outputs of two commands
diff <(command1) <(command2)
```

This process is called process substitution and it is almost never useful, but when it is it can save up a great amount of time.

If you are intrestes in some advanced features of bash redirections you can go to the [REDIRECTION](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) section of `man bash`.
