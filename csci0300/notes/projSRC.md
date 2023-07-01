## 实验简介
### software
这个实验是timemachine类型的实验，希望通过安装模拟器等其它方式来使用以前的软件，读取以前的文本。

但说实在的这个实验真的相当麻烦，让人发掘其实更多时候还是重构和复刻给人带来的喜悦更多一些。

首先需要意识到文件内comments的存在，利用comments内的提示，采用ctrl F5功能来利用TEXT_IN和TEXT_OUT方式，将文本以DOS-TXT的形式存储下来。

这个存储方式是以ASCII码来做的，现在依旧在使用，因此能够在现如今的机子中发挥功效。

最终得到的文本如下所示，不过也可能是我处理漏了一些步骤，comments的信息并没有一并存储下来。
```
                    CSCI0300 WordPerfect 5.1 Guide

      This is an introductory reference manual for WordPerfect 5.1,
created by the CSCI0300: Fundamentals of Computer Systems staff.




WordPerfect 5.1 originated as a
DOS word processor in 1979,
developed by BYU CS Professor Alan Ashton and graduate student
Bruce Bastian for the Data General minicomputer system. WordPerfect
2.20 for IBM PC was released in 1982, with DOS support released in
1983 with WordPerfect 3.0. Its features were significantly more
advanced than its competitor WordStar, and eventually displaced it
to become the dominant player in the market.

Unfortunately, due to a delayed release on Microsoft Windows, the
release of MS Word quickly displaced WordPerfect to become the
dominant word processor, and the rest is history. WordPerfect does
remain a favorite among lawyers, due to its Reveal Codes feature,
rapid editing capabilities once the keystrokes were mastered, and
formatting prowess, especially in creating consistent and strictly-
formatted documents like Pleadings.




To start, F3 opens up the Help menu. Use this to search for any
commands that you may desire; if there's a feature you're looking
for, you'll probably find it here. Indeed, the following keystrokes
are all readily available in the Help menu; they are listed here
merely for your convenience.

Common text formatting commands:
      - F6: Bold
      - Ctrl-F8,2,4: Italics
      - F8 <OR> Ctrl-F8,2,2: Underline
      - Ctrl-F8,3: Normal
      - F4: Indent Left
           - Shift-F4: Indent Left and Right
      - Shift-F6: Center Text
      - Shift-F8: Headers/Footers/lots more!
      - Ctrl-F7: Footnotes

Search and Replace:
      - F2: Forward Search
      - Shift-F2: Reverse Search
      - Alt-F2: Replace

Save/Exit:
      - F10: Save Document to Disk
      - F7: Save and Exit

Explore Directory:
      - F5: Look around
           - Press Enter, then type in the directory to explore
           - When done, use F7 to exit

Preview:
      - Shift-F7,6: Preview Document

Macros:
      - Ctrl-F10: Macros Editor
      - Ctrl-PgUp: Macros List
      - Alt-F10: Execute Macro 

Graphics:
      - Alt-F9: Graphics
           - Alt-F9,3: Box
           - Alt-F9,5: Line




This may help with navigating around WordPerfect. We hope you
enjoyed this journey to the past!




                              ASSIGNMENT
Finding this documentation helpful, you want to save it for future
use. Unfortunately, opening DOSBox, mounting your C: drive, then
opening WordPerfect is a little too cumbersome; additionally, after
this process you begin to realize the fragility of emulation
software, and thus desire a more accessible, universal format:
ASCII.

      1) Convert this document into ASCII, and save it into a
      file named "cs300ref.txt". You may find this helpful.

In addition, we have included the following code: 

                           CSCI0300ROCKS!!!

      2) In an file named "secret-code.txt", write the entirety of
      the code, including the Reveal Codes of the entire line.






CSCI0300 Inc.
Copyright (c) 1990
```
### hardware
尝试跑了一下，这些老游戏也太抽象了.....
