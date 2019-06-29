---
title: "How I use Vim for everything"
date: 2019-06-02T14:28:15+02:00
categories: ["TODO"]
tags: ["TODO"]
authors: ["Rob"]
draft: true
---

Often times I get asked what IDE to I use, and every time I answer "I use Vim" people either look at me as if I was a ninja or just completely crazy (which I am, but not for this reason).

This post purpose is to show that Vim can just be as comfortable to use as most modern IDEs and it can do that without carrying the webcam driver for the XBox and a JavaScript engine.

# Vanilla
Most of the stuff I use in Vim doesn't come from plugins, it is the default functionality that comes pre-installed with the editor. The only issue is that to use those features you need to know they exist. This is what makes Vim learning curve very steep and the first impression is usually "How the fork do I exit this thing?".

Vim has an amazing help, but to use it you usually need to know what you're looking for, so here are some suggestions.

## Basic and less basic configuration
```vim
TODO

```

## Windows
In vim you can split your screen with vertical and horizontal splits. It fairly easy: instead of opening a new file with `:e file` you do it with `:vert spl file` or just `:spl file`. Then to switch window you can use `Ctrl+W` and a movement key (hjkl). To rearrange windows it is still `Ctrl+W` and uppercase movement keys(HJKL).

## Terminal emulator
You can have a terminal emulator in a vim buffer(so windows, tabs, background views etc.). You just have to type `:term`. Splits apply to this command so you also have `:vert term`. This allows you to move the terminal around and go to "normal mode" in that command line, so you can easily copy-paste from the terminal to vim and vice versa with just vim commands. Since a terminal is still a buffer you can send it to the background and go back to it whenever you want.

## Buffers
Every thing you opened in vim is a buffer, and you can quickly switch between them with `:buf file` (supports fuzzy match and tab-completion) or list them with `:buffers` and press the number corresponding to the one you want.

## Multi Replace
Let's say you want to make this change:
```
// Before:
I want a foo to foo my foo but not my foos.
// After:
I want a bar to bar my bar but not my foos.
```
In vim you just go to the first "foo", then you press `*` or `#` (search all occurrences of the word under the cursor, then go to prev or next) and do `cgn` which you can memorise as change(c) and go(g) to next(n). Now you are back in insert mode, you type "bar" and you press `Esc`. If you now press `.` it will repeat the last action: find next/prev occurrence of "foo", replace it with "bar". You can skip a finding with next(n).

## Tabs and mouse
Tabs can be opened with `:tabe file` (tab-edit) or `:tab term` for a terminal. Tabs are mostly frowned upon as they can be just replaced with buffers in a faster way, but if you are very visual you can then switch to other tabs with `:tab(prev|next|last|first)` or just clicking with the mouse (requires `:set mouse=a`).

## Autocompletion
TODO

## Spellcheck
You can enable spellchecking with `:set spell`. You can then move around misspelled words with `[s` and `]s` and get suggestions with `z=`. The spell command is so featureful I strongly suggest to give `:help spell` a read.

If vim recognizes the syntax for your language it is possible to have spellcheck only on comments and string literals and never in code.

## Tags
TODO

## Visual Block
I feel like this mode is broadly underestimated. Let's see you have to change this code:
```go
// Before:
a := []string{
	"say",
	"zed",
	"keg",
	"dis",
	"gap",
	"eva",
}

// After
a := map[string]string{
	"say":"Say",
	"zed":"Zed",
	"keg":"Keg",
	"dis":"Dis",
	"gap":"Gap",
	"eva":"Eva",
}
```
You can achieve this by positioning on the first quote before "say", then `Ctrl+v` + `5j`, which enters visual block mode and jumps(j) 5 lines below. Then you can move right 4 characters with `4l` and yank(y) the block. You can then paste(p) it and it will appear next to the text that is already there. Now that you have two duplicated columns you can do a visual column selection (again `Ctrl+v`) and move down, this allows you to type ":" and when you press `Esc` the colon will appear on all selected characters.

To uppercase the words initials it is the same: visually select the column and switch to uppercase(U).

## Fuzzy Find
TODO

## The leader key and auto boilerplate for doc

## Abbreviations, AKA snippets
Let's say you write go and you find yourself typing `if err != nil {↓` many times a day, you can shorten this by adding `:iabbrev errn if err != nil {<CR>` (CR stands for Carriage Return) and then when you type the word "errn" the full snippet will appear instead. You can do this for just go files if you want: `autocmd FileType go iabbrev errn if err != nil {<CR>`.

This is also very useful for common typos: I use this for automatically correcting words that I often misspell.

## Advanced motion
### 'Till
TODO

### Blocks
Let's say that you have to factor-out some code from a for loop to it's own function because it is otherwise too complex to read:
```go
func old(){
	for i:=0; i<N; i++ {
		// Code here
	}
}

func new(){
	// Code should end up here
}
```

In most editors you'd need to move your hand away from the keyboard to the mouse and tilt the other one towards the `Ctrl` key. In Vim you can position your cursor anywhere in the `for` body and type
`di{` in normal mode. This will delete(d) code inside(i) the braces({). Then you can just jump to the `new` function and paste(p). This doesn't just work for braces, it can be done with words, quotes, all parentheses and most otherwise semantic blocks.

If you also want to bring the parentheses with you you can do `da{` which is delete(d) code around(a) braces({).

This might seem like a small thing, but I use it tens of times a day.

### The position stack
TODO local and global

### Go to definition
TODO

## Plugins
If you are not satisfied with vanilla, you can spice it up with some plugins. I don't use many but these are my favorite ones:
TODO
