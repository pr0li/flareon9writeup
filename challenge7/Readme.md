# Solving #flareon9 challenge 7 (anode) with just notepad++ and regex

This challenge is a **node.js** script that has been compiled into an executable file. You'll notice right away that the executable is a very big file (54 MB). If you open the binary in a text editor (but please don't do that in notepad :D) and go to the end of the file, you'll be able to see the js script that is executed.

There are thousands of lines in the js code and a huge switch statement with a ton of different cases. In other words, lovely **control flow obfuscation**. Before the switch statement, the program takes an input string from the user, then several math operations are performed to the individual characters of the input, and finally the result is compared to some hardcoded values at the end of the script.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/01.png "First look at the script at the end of the binary file")

All these operations are reversible: addition, subtraction, XORing; therefore it is possible to apply all the operations in reverse order to the expected output to obtain the correct input.

The first thought that comes to mind is to extract the js code and edit it or run it outside of this binary file. However, the program uses several **Math.random** and **Math.floor** functions whose output is not what we would expect in a normal js environment. Somehow these libraries have been tampered with before compiling them into the executable. You can see a clear example in the next image: this control statement will always make the program exit outside of this modified environment (because 1 is always true).

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/02.png "Example of a control statement with altered result")

So for my solution, I will use the executable itself (with some modifications) to show me the correct order of math operations that I have to do. I will not try to figure out what was modified in the libraries, I want to solve this fast and move on to the next challenge.

For starters, I will go to line **487829** (see the image below) and cut from there until the end of the file to a separate text file so that I don't have to deal with such a big file. I will do this step in a hex editor like HxD, because there are some null characters at the end of the file that I don't want to lose.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/03.png "Start of the script where we will do replacements")

What I want to do is to let all the math operations execute, but at the same time to use **console.log()** to write which operations are being executed so I can then reverse the order after. And I want to keep the size of the script exactly the same as it was originally, so if I add 10 characters, I'll have to remove 10 characters somewhere else. The image below shows that I have 9571 lines and **321401** characters, so I'll always try to go back to that number before copying this back to the executable. I also have to be careful to select Unix (LF) and not Windows (CR LF) in my text editor.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/04.png "Original size of the script")

The first thing that I'm going to do is to define an alias for **console.log()**, so that I can use less characters every time I call **console.log()**.

```javascript
c=console.log;
c('Hello, world!');
```

I will add this to the start of the script.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/05.png "Added alias for console.log to the script")

Then I'm going to start replacing every line that does some operation to a character in our input for **"c()"** to log the operation. I don't care about the actual result of the operation at this point, but I do care about the **Math.floor** and **Math.random** being executed because they need to be executed in the right order to produce the right results.

First I'll replace all lines that perform operations on the characters, and that involve the altered libraries. The next image shows a *"before"* and *"after"* of one of these replacements.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/06.png "Before and after the replaced lines")

The regex to do this is:

**Find what:**
```
          (b\[[0-9]{1,2}\] [\-\+\^]= )\(?(b\[.+ \+ )(Math.+256\))(\) & 0xFF)?;
```
**Replace with:**
```
 c\('\1\2'\+\3\);
```
Note that I'm still executing the library calls but only logging the result. I'm also replacing all the spaces at the start with just one space (to make up for the characters that I added).

Also note that some of the operations have **"& 0xFF"** at the end and some of them don't (but they have a separate line to do the binary AND). So I'm ignoring the AND operation, and I'm not printing all the other individual lines with the AND operations. This will be useful later on when I have to reverse the operations, because it's easier to reverse a list of one-liners and I can always add the AND lines after.

Next, I want to print all the other operations that don't use the altered libraries.

Regex:
**Find what:**
```
          (b\[[0-9]{1,2}\] [\-\+\^]= )\(?(b\[.+ \+ [0-9]{1,})(\) & 0xFF)?;
```
**Replace with:**
```
 c\('\1\2'\);
```
So I get something like in the next image.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/07.png "Script after more replacements")

I removed more characters than I added, so the total count is lower than it should be. I'll just add spaces somewhere at the end. The next image shows that now we have the right size.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/08.png "We have the same size after the replacements")

Then I'll overwrite the original executable with the new script, using HxD. After that, I run the patched executable with the output that can be seen in the next image.

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/09.png "Execution of the patched binary")

The complete output is in [output.txt](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/output.txt):

Then I can reverse the order in notepad++ with *Edit>Line Operations>Reverse Line Order* (I have notepad++ v8.4.7). Also, I need to add those **b[X] &= 0xFF** that I ignored before. I can do that with a regex as well.

Regex:
**Find what:**
```
(b\[[0-9]{1,2}\])( [\-\+\^]= .+)\n
```
**Replace with:**
```
\1\2\n\1 &= 0xFF\n
```

Finally, I will replace all **"+="** for **"-="** and the opposite.

Regex:
**Find what:**
```
\+=
```
**Replace with:**
```
@=
```

**Find what:**
```
\-=
```
**Replace with:**
```
\+=
```

**Find what:**
```
@=
```
**Replace with:**
```
\-=
```

I can put this in a Python script (I guess that makes the title clickbait :D) to obtain the flag.

The script is in [solver.py](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/solver.py).

Cropped snippet here:
```python
b = [106, 196, 106, 178, 174, 102, 31, 91, 66, 255, 86, 196, 74, 139, 219, 166, 106, 4, 211, 68, 227, 72, 156, 38, 239, 153, 223, 225, 73, 171, 51, 4, 234, 50, 207, 82, 18, 111, 180, 212, 81, 189, 73, 76]

b[39] -= b[18] + b[16] + b[8] + b[19] + b[5] + b[23] + 36
b[39] &= 0xFF
b[22] -= b[16] + b[18] + b[7] + b[23] + b[1] + b[27] + 50
b[22] &= 0xFF
b[34] -= b[35] + b[40] + b[13] + b[41] + b[23] + b[25] + 14
b[34] &= 0xFF
b[21] -= b[39] + b[6] + b[0] + b[33] + b[8] + b[40] + 179
b[21] &= 0xFF
b[11] += b[32] + b[8] + b[9] + b[34] + b[39] + b[19] + 185
b[11] &= 0xFF
b[19] ^= b[0] + b[35] + b[14] + b[30] + b[21] + b[33] + 213
b[19] &= 0xFF
b[40] -= b[13] + b[3] + b[43] + b[31] + b[22] + b[25] + 49
b[40] &= 0xFF
b[17] ^= b[41] + b[14] + b[43] + b[6] + b[7] + b[28] + 196
b[17] &= 0xFF
b[38] -= b[20] + b[30] + b[31] + b[8] + b[37] + b[33] + 54
b[38] &= 0xFF
b[19] ^= b[3] + b[30] + b[17] + b[15] + b[13] + b[18] + 241
b[19] &= 0xFF
#
# CROPPED LINES HERE, BUT THE FULL SCRIPT IS IN SOLVER.PY
#
print("".join(chr(c) for c in b))
```

![alt text](https://github.com/pr0li/flareon9writeup/blob/main/challenge7/img/10.png "Running the script to obtain the flag")

## Much code, great success!
