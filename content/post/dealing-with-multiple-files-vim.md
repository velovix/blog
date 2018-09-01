---
title: "Dealing With Multiple Files in VIM"
date: "2016-07-28T14:05:00-07:00"
draft: false
---

The VIM learning curve is legendarily steep. As if the modal editing wasn't
tricky enough, a lot of basic editor features that most users take for granted
appear at first to be absent completely, like syntax highlighting,
autocomplete, and, as we'll be talking about here, file navigation.

Some of these issues have simple solutions (:syntax on, for instance), but it
turns out that dealing with multiple files in VIM is a complicated subject with
a lot of solutions, some better than others. This post explores the various
options available, in an order that I believe most VIM beginners end up finding
them in.

## Option 1: Tabs

When researching this issue, new users are bound to find some mention of tab
support in VIM. They feel relieved by the familiar terminology and immediately
start using tabs like how tabs are used in editors like Atom and IntelliJ IDEA,
namely one buffer per tab. However, the seeming glint of familiarity is but a
trap for new users, as tabs in VIM are not like tabs in those editors, and the
new user will soon find tabs to be a very clunky solution to this problem.

The problem here is that new users intuitively think of tabs as places for
single files, but in VIM, they're closer to what an IDE might call a "layout".
Users expect tabs to be contained in windows, but in fact, it's the opposite.
That makes tabs work very poorly as a container for a single file. This makes
tabs a *bad option.*

## Option 2: Raw Buffers

The simplest solution to dealing with multiple files is to simply have VIM open
multiple files. Early on, I would open up a project by jumping to its directory
and running `vim *.c` to open every C file, or `vim *.go` to open every Go
file. Then I "simply" scrolled through each buffer with repeated calls to `bn`
(buffer next) and `bp` (buffer previous). If I needed to open up another file,
I would use the :e command followed by the path to the file. If I forgot what
it was called, then I had to go consult `ls` or a file manager. That was
roughing it.

This is closer to a right solution than tabs are, and I suspect that some
hardcore no-frills VIM users still do use this pattern. However, if you ask me,
using buffers on their own gets tiresome, and I much prefer using a tool to
help me out.

## Option 3: Netrw and Buffers

Netrw is VIM's built-in file manager. When invoked, it takes over the buffer it
is run in, and replaces the buffer with whatever file you choose. Netrw is not
a bad option, but it's not a great option either.  It's notoriously buggy and a
lot of the bugs that appear in its less used features, like tree view, don't
end up getting fixed because so few people use them. Personally, only
[one bug](https://github.com/tpope/vim-vinegar/issues/21) ever really bothered me, but
[other people](https://www.reddit.com/r/vim/comments/22ztqp/why_does_nerdtree_exist_whats_wrong_with_netrw/cgs4aax)
seem to have a worse time than I. Your milage may vary.

## Option 4: NERDTree and Buffers

NERDTree is another tool that makes beginner VIM users feel more at home. It
opens up a file tree as a split window on the left side of the workspace, just
like an IDE would. Users can jump to that window and select their file in a
very user-friendly way.

Personally, I think this is a very good option for new VIM users. The
familiarity is comforting, and unlike tabs, it's not a trap. Some more advanced
users use NERDTree because it doesn't tend to have the bugs that Netrw does.
Hardcore VIM-ers don't tend to like NERDTree because they find it bloated,
though those tend to be the kind of people that think most any plugin is
unnecesary bloat. I don't think that's a very productive attitude for new
VIM-ers to have.

I've moved away from the "project drawer" style of NERDTree and most every IDE,
because I find it isn't a very good use of a space, but I know a lot of people
like project drawers and if you, dear reader, are one of those people, then
NERDTree is right up your alley.

## Option 5: vim-vinegar and Buffers

The vim-vinegar plugin is for people like me that don't mind Netrw's bugs and
like that it takes over the whole window, but could benefit from a better user
experience. The plugin assigns `-` as the Netrw hotkey and automatically points
it to the directory of the file you're currently editing. I find this to be
what I want 99.9% of the time and is, on its own, a huge improvement to Netrw.
It also does a whole bunch of smaller things that I also appreciate.

vim-vinegar is the file explorer setup that I personally use, but it might not
fit everyone's taste.  I encourage everyone to give it a try.

## Addons

Sometimes, consulting a file explorer isn't always the way you want to switch
buffers. If you need to switch between two separate files a lot, that can get
pretty tedious. Here are some plugins that I have used to make that process
easier.

### minibufexpl.vim

Beyond having a beautiful name that rolls right off the tongue, minibufexpl.vim
displays a list of open buffers as a small window at the top of the workspace
with their corresponding buffer numbers. This makes it easy to look up the
number of the buffer you want, and jump to it with `:#b` where # is the buffer
number.

I ended up not sticking with this plugin for a few reasons.

1. I'm not into project drawers. (see Option 4: NERDTree and Buffers)

2. Some plugins leave behind buffers that clutter up the list. minibufexpl.vim
   doesn't seem to know how to filter these out.

3. minibufexpl.vim is less useful the more files you have open. The longer that
   list gets, the longer it takes for you to find the file you want to jump to.
   It requires you to constantly clean up buffers you're no longer using. Some
   might call that best practice. I call it annoying.

### ctrlp.vim

This plugin will feel immediately familiar to users of IntelliJ IDEA who like
the CTRL+Shift+P feature. ctrlp.vim is a fuzzy finder for files in and past the
directory you started VIM in.  I find this to be an excellent complement to
vim-vinegar for files that are far away directory-wise but that you know the
name of.

The search algorithm ctrlp.vim uses isn't quite as strong as IntelliJ's. For
instance, you can't type in an acronym and find a long Java class file, but
it's good enough to be very useful.

Admittedly, ctrlp.vim is a bit of a project drawer plugin, but it doesn't
linger and I find it useful enough to forgive it for that sin. :)
