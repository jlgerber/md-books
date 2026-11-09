# General

## Parts 

### Buffer

A representation of a file in memory. You manipulate text in buffers. 

### Window 

A view of a buffer. You can open multiple windows pointing at different parts of the same buffer. 


## Modes 

Vim is a modal editor; you are always in one of a number of different modes. There area four common modes:

### Normal

You can move your cursor around by typing.

#### Movements

Key | Description 
--- | --- 
j   | Move downward 
k   | Move upward 
l   | Move right 
h   | Mode left 
w   | Move one word forward 
b   | Move one word back

 These keys are all *motions*. Motions can be preceded with a count or number. This is the number of times to execute the motion. So, `5w` takes you forward 5 words.

Key | Description 
--- | ---
d   | delete prefix 
d#m | In general, delete + optional number + motion
dw  | delete the word that you are on 
d$  | delete to the end of the line 
dd  | delete the current line

#### Copying and Pasting Text

You can *yank* text by using the `y` key and following it with an optional count and motion

Key | Description
--- | --- 
yy  | Copy the current line 
yw  | copy the current word under mouse 

You paste copied text using the `p` command.

### Insert 

You can enter text by typing.

#### Entering Insert Mode 

From Normal Mode...
- Press *i* to go into insert mode. Cursor will move to the left
- Press *a* to go into insert mode. Cursor will move to the right

#### Exiting Insert Mode

You can exit Insert mode by pressing `ESC`. 

### Visual Line 

Highlighting selections.

#### Copy / Paste 

Yanking works in Visual Line mode. You simply press `y` and highlighted text will be copied to the buffer. Subsequently you can press `p` to paste the text to the buffer at the current cursor's position.

### Command Mode 

If you press colon (':') in normal mode, this puts you in command mode. Your cursor appears in the lower right hand of the footer. From here you can issue common commands.

- **:w** - save the current file (buffer)
- :wq - save the current file and quit the buffer (and potentially vim)

## Horizontal Motion

### Moving to the Beginning or End of the Line

Key | Description
--- | ---
0   | Cursor goes to the beginning of the line (0th character)
_   | Cursor goes to the beginning of the line
^   | Cursor goes to beginning of text of the current line 
$   | Cursor goes to end of the line 
I   | Move to the beggining of the line, in insert mode 
A   | Move to the end of the line but in insert mode


### Moving To a Specific Character

Key | Description 
--- | --- 
f   | Move forward to the provided subsequent character (eg `f(` to move to the next parens )
t   | Move up to but not on the specified character
F   | Move *backwards* to the next provided character
T   | Move *backwards* up to the next provided character

Once in the find mode, you can use `,` to go back and `;` to go forward 

## Vertical Motion 

### Opening new lines 

Key | Description 
--- | --- 
o   | Make a new line and go into insert mode 
O   | Make a new line above your cursor and go into insert mode

