# Vim Commands Quick Reference Guide

## Modes
```text
i             insert before cursor
I             insert at beginning of line
a             append after cursor
A             append at end of line
o             open new line below
O             open new line above
Esc           return to normal mode
v             visual mode
V             visual line mode
Ctrl + v      visual block mode
:             command-line mode
```

## Save, Quit, and Files
```text
:w            save file
:w <file>     save as file
:q            quit
:q!           quit without saving
:wq           save and quit
:x            save and quit if changed
:e <file>     open file
:r <file>     read file into current buffer
:set number   show line numbers
:set nonumber hide line numbers
```

## Movement
```text
h             left
j             down
k             up
l             right
w             next word
b             previous word
e             end of word
0             beginning of line
^             first non-blank character
$             end of line
gg            first line
G             last line
<n>G          go to line number
Ctrl + f      page forward
Ctrl + b      page back
%             jump to matching bracket
```

## Editing
```text
x             delete character
dd            delete line
dw            delete word
d$            delete to end of line
yy            copy line
yw            copy word
p             paste after cursor
P             paste before cursor
u             undo
Ctrl + r      redo
.             repeat last change
r<char>       replace one character
cw            change word
cc            change line
C             change to end of line
```

## Search and Replace
```text
/text         search forward
?text         search backward
n             next match
N             previous match
*             search word under cursor forward
#             search word under cursor backward
:noh          clear search highlight
:%s/old/new/g replace all matches in file
:%s/old/new/gc replace all matches with confirmation
:s/old/new/g  replace matches on current line
```

## Visual Selection
```text
v             select characters
V             select lines
Ctrl + v      select block
y             copy selection
d             delete selection
>             indent selection
<             unindent selection
=             auto-format selection
```

## Buffers, Windows, and Tabs
```text
:ls           list buffers
:b <n>        switch to buffer number
:bn           next buffer
:bp           previous buffer
:bd           close buffer
:split        horizontal split
:vsplit       vertical split
Ctrl + w w    move to next window
Ctrl + w h    move to left window
Ctrl + w j    move to lower window
Ctrl + w k    move to upper window
Ctrl + w l    move to right window
:tabnew       open new tab
gt            next tab
gT            previous tab
```

## Marks, Jumps, and Macros
```text
m<char>       set mark
'<char>       jump to mark line
`<char>       jump to exact mark position
Ctrl + o      jump back
Ctrl + i      jump forward
q<char>       start recording macro
q             stop recording macro
@<char>       run macro
@@            repeat last macro
```

## Useful Commands
```text
:help <topic> open help
:syntax on    enable syntax highlighting
:set paste    paste without auto-indent changes
:set nopaste  return to normal paste behavior
J             join current line with next line
~             toggle character case
gUiw          uppercase current word
guiw          lowercase current word
```
