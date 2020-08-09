## 正则表达式

>  `^` 表示一行的开头。如：`/^#/` 以#开头的匹配。
> 
>  `$` 表示一行的结尾。如：`/}$/` 以}结尾的匹配。
>
>  `\<` 表示词首。 如：`\<abc` 表示以 abc 为首的詞。
> 
>  `\>` 表示词尾。 如：`abc\>` 表示以 abc 結尾的詞。
> 
>  `.` 表示任何单个字符。
> 
>  `*` 表示某个字符出现了0次或多次。
> 
>  `[ ]` 字符集合。 如：`[abc]` 表示匹配a或b或c，还有 `[a-zA-Z]` 表示匹配所有的26个字符。如果其中有^表示反，如 `[^a]` 表示非a的字符



## 三剑客

- grep擅长查找功能，sed擅长取行和替换。awk擅长取列。

  -    grep 更适合单纯的查找或匹配文本
  -    sed 更适合编辑匹配到的文本
  -    awk 更适合格式化文本，对文本进行较复杂格式处理

### grep

- grep可用于shell脚本，因为grep通过返回一个状态值来说明搜索的状态，如果模板搜索成功，则返回0，如果搜索不成功，则返回1，如果搜索的文件不存在，则返回2。我们利用这些返回值就可进行一些自动化的文本处理工作。

- grep 常用的参数如下：

  - -A<行数 x>：除了显示符合范本样式的那一列之外，并显示该行之后的 x 行内容。
  - -B<行数 x>：除了显示符合样式的那一行之外，并显示该行之前的 x 行内容。
  - -C<行数 x>：除了显示符合样式的那一行之外，并显示该行之前后的 x 行内容。
  - -c：统计匹配的行数
  - **-e ：实现多个选项间的逻辑or 关系**
  - **-E：扩展的正则表达式**
  - -f 文件名：从文件获取 `PATTERN` 匹配
  - -F ：相当于`fgrep`
  - -i --ignore-case #忽略字符大小写的差别。
  - -n：显示匹配的行号
  - -o：仅显示匹配到的字符串
  - -q： 静默模式，不输出任何信息
  - -s：不显示错误信息。
  - **-v：显示不被 pattern 匹配到的行，相当于[^] 反向匹配**
  - -w ：匹配 整个单词

  前三个 A、B、C 参数很容易理解，举个栗子，假设我们有一个文件，文件名是 test，内容是从 1 到 9，每个数字一行：

  ```bash
  ➜ grep -A2 7 test
  7
  8
  9
  ```

  `-A2 7` 的效果就是找到 7 ，然后输出 7 后面两行。

  同理，`-B2 7`和`-C2 7`就是找到 7 ，然后分别输出 7 前面两行和前后两行：

  ```bash
  ➜ grep -B2 7 test
  5
  6
  7
  
  ➜ grep -C2 7 test
  5
  6
  7
  8
  9
  ```

  继续，假设我们有个名叫 test 的文件内容如下：

  ```bash
  ➜ cat test
  aaaa
  bbbbbb
  AAAaaa
  BBBBASDABBDA
  ```

  `grep -c`命令的作用就是输出匹配到的行数，比如我们想找包含`aaa`的有几行，一眼就能看出来有两行，第一行和第三行都包含：

  ```bash
  ➜ grep -c aaa test
  2
  ```

  `grep -e`命令是实现多个匹配之间的`或`关系，比如我们想找包含`aaaa`或者`bbbb`的，显然应该返回第一行和第二行：

  ```bash
  ➜ grep -e aaaa -e bbbb test
  aaaa
  bbbbbb
  ```

  `grep -F`相当于`fgrep`命令，就是将`pattern`视为固定字符串。比如搜索`'aa*'`不带`-F`和带上，区别如下：

  ```bash
  ➜ grep 'aa*' test
  aaaa
  AAAaaa
  
  ➜ grep -F 'aa*' test
  ```

  可以看到第二次就找不到了，因为搜索的是 `aa*`这个字符串，而不是正则表达式。

  `grep -f 文件名`的使用方法是把后面这个文件里的内容当做`pattern`。比如我们有个文件，名字是 grep.txt，然后内容是`aa*`，使用方法如下：

  ```bash
  ➜ grep -f grep.txt test
  aaaa
  AAAaaa
  ```

  实际上等同于`grep 'aa*' test`

  `grep -i --ignore-case`作用是忽略大小写。

  `grep -n`显示匹配的行号，就是多显示了个行号，不用细说。

  `grep -o`仅显示匹配到的字符串，还是用刚才的`aa*`距离，之前显示的都是匹配到的字符所在的整行，这个命令是只显示匹配到的字符：

  ```java
  ➜ grep -o 'aa*' test
  aaaa
  aaa
  ```

  `grep -q`不打印匹配结果。刚看到这个我疑惑了半天，让你搜索字符串，你不给我结果那有啥用？然后发现还有一条很多教程没说：如果有匹配的内容则立即返回状态值 0。所以一般用在`shell`脚本中，在 `if` 判断里面。

  `grep -s`不显示错误信息，不解释。

  `grep -v`显示不被匹配到的行，相当于`[^]`反向匹配，最常见的还是用在查找线程的命令里，有时候会打印`grep`线程，可以再加上这么一个去除自己：

  ```bash
  ➜ ps -ef|grep Typora
    501 91616     1   0 五11上午 ??        13:39.32 /Applications/Typora.app/Contents/MacOS/Typora
    501 14814 93748   0  5:33下午 ttys002    0:00.00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn Typora
  
  ➜ ps -ef|grep Typora|grep -v grep
    501 91616     1   0 五11上午 ??        13:39.32 /Applications/Typora.app/Contents/MacOS/Typora
  ```

  可以看到第二次就没有打印`grep`线程自身

  `grep -w`匹配整个单词，只有完全符合`pattern`的单次才会匹配到：

  ```bash
  ➜ grep aaa test
  aaaa
  AAAaaa
  
  ➜ grep -w aaa test
  ```

  可以看到第二次结果为空，因为没有`aaa`这个单词。

  关于正则的高级用法就不再深入研究了，改日再统一整理。

### sed

- 参数

  - -e表示多点编辑

- 动作说明：

  - sed 后面接的动作，请务必以 '' 两个单引号括住

  >  a ：新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)
  >
  >  c ：取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行！
  >
  >  d ：删除
  >
  >  i ：插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
  >
  >  p ：打印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行
  >
  >  s ：取代，可以直接进行取代的工作。例如sed 's/要被取代的字串/新的字串/g'
  >
  >  
  >
  >  这里用双引号把整个表达式括起来也可以，还方便处理带空格的字符。
  >
  >  ```shell
  >  // sed /Linux/i\newline test是在所有匹配到Linux的行前面插入
  >  sed -e /Linux/a\newline test`等效于`sed "/Linux/a newline" test
  >  ```

- > 删除的字符是`d`，用法跟前面也很相似，就不赘述，例子如下：
  >
  > ```bash
  > ➜ sed '/Linux/d' test  
  > ```
  >
  > $ sed -e 4a\newline testfile #使用sed 在第四行后添加新字符串  
  >
  > 
  >
  > sed '2,5d' testfile  # 将第 2~5 行删除
  >
  > 
  >
  > 如果是要增加两行以上，在第二行后面加入两行字
  >
  > 每一行之间都必须要以反斜杠『 \ 』来进行新行的添加
  >
  > sed '2a Drink tea or ......\ >drink beer ?'
  
- $ sed "s/my/Hao Chen's/g" pets.txt

  - 把其中的my字符串替换成Hao Chen’s，下面的语句应该很好理解（s表示替换命令，/my/表示匹配my，/Hao Chen’s/表示把匹配替换成Hao Chen’s，**/g 表示一行上的替换所有的匹配**）

  - 注意：如果你要使用单引号，那么你没办法通过\’这样来转义，就有双引号就可以了，在双引号内可以用\”来转义。

  - > 上面的sed并没有对文件的内容改变，只是把处理过后的内容输出，如果你要写回文件，你可以使用重定向
    >
    > - $ sed "s/my/Hao Chen's/g" pets.txt > hao_pets.txt
    > - 或使用 -i 参数直接修改文件内容：
    > - $ sed -i "s/my/Hao Chen's/g" pets.txt

- > 在每一行最前面加点东西：
  >
  > 
  >
  > $ sed 's/^/#/g' pets.txt
  >
  > 
  >
  > \#This is my cat
  >
  > \#  my cat's name is betty
  >
  > \#This is my dog
  >
  > \#  my dog's name is frank
  >
  > \#This is my fish
  >
  > \#  my fish's name is george
  >
  > \#This is my goat
  >
  > \#  my goat's name is adam
  >
  > 
  >
  > 在每一行最后面加点东西：
  >
  > 
  >
  > $ sed 's/$/ --- /g' pets.txt
  >
  > 
  >
  > This is my cat ---
  >
  >   my cat's name is betty ---
  >
  > This is my dog ---
  >
  >   my dog's name is frank ---
  >
  > This is my fish ---
  >
  >   my fish's name is george ---
  >
  > This is my goat ---
  >
  >   my goat's name is adam --
  >
  > 
  >
  > 下面的命令只替换第3到第6行的文本。
  >
  > $ sed "3,6s/my/your/g" pets.txt
  >
  > 
  >
  > 只替换每一行的第一个s：
  >
  > $ sed 's/s/S/1' my.txt
  >
  > 
  >
  > 只替换每一行的第二个s：
  >
  > $ sed 's/s/S/2' my.txt
  >
  > 
  >
  > 只替换第一行的第3个以后的s：
  >
  > $ sed 's/s/S/3g' my.txt

#### sed的命令

- > ##### N命令
  >
  > 先来看N命令 —— 把下一行的内容纳入当成缓冲区做匹配。
  >
  > 下面的的示例会把原文本中的偶数行纳入奇数行匹配，而s只匹配并替换一次，所以，就成了下面的结果：
  >
  > 
  >
  > $ sed 'N;s/my/your/' pets.txt
  >
  > This is your cat
  >
  >   my cat's name is betty
  >
  > This is your dog
  >
  >   my dog's name is frank
  >
  > This is your fish
  >
  >   my fish's name is george
  >
  > This is your goat
  >
  >   my goat's name is adam
  >
  > 
  >
  > 也就是说，原来的文件成了：
  >
  > 
  >
  > This is my cat\n  my cat's name is betty
  >
  > This is my dog\n  my dog's name is frank
  >
  > This is my fish\n  my fish's name is george
  >
  > This is my goat\n  my goat's name is adam
  >
  > 
  >
  > 这样一来，下面的例子你就明白了，
  >
  > 
  >
  > 
  >
  > 
  >
  > $ sed 'N;s/\n/,/' pets.txt
  >
  > This is my cat,  my cat's name is betty
  >
  > This is my dog,  my dog's name is frank
  >
  > This is my fish,  my fish's name is george
  >
  > This is my goat,  my goat's name is adam

  

- > ##### a命令和i命令
  >
  > a命令就是append， i命令就是insert，它们是用来添加行的。如：
  >
  > 
  >
  > \# 其中的1i表明，其要在第1行前插入一行（insert）
  >
  > $ sed "1 i This is my monkey, my monkey's name is wukong" my.txt
  >
  > This is my monkey, my monkey's name is wukong
  >
  > This is my cat, my cat's name is betty
  >
  > This is my dog, my dog's name is frank
  >
  > This is my fish, my fish's name is george
  >
  > This is my goat, my goat's name is adam
  >
  > 
  >
  > \# 其中的1a表明，其要在最后一行后追加一行（append）
  >
  > $ sed "$ a This is my monkey, my monkey's name is wukong" my.txt
  >
  > This is my cat, my cat's name is betty
  >
  > This is my monkey, my monkey's name is wukong
  >
  > This is my dog, my dog's name is frank
  >
  > This is my fish, my fish's name is george
  >
  > This is my goat, my goat's name is adam
  >
  > 我们可以运用匹配来添加文本：
  >
  > 
  >
  > \# 注意其中的/fish/a，这意思是匹配到/fish/后就追加一行
  >
  > $ sed "/fish/a This is my monkey, my monkey's name is wukong" my.txt

  

- > ##### c命令
  >
  > c 命令是替换匹配行
  >
  > $ sed "2 c This is my monkey, my monkey's name is wukong" my.txt
  >
  > This is my cat, my cat's name is betty
  >
  > This is my monkey, my monkey's name is wukong
  >
  > This is my fish, my fish's name is george
  >
  > This is my goat, my goat's name is adam
  >
  > 
  >
  > $ sed "/fish/c This is my monkey, my monkey's name is wukong" my.txt
  >
  > This is my cat, my cat's name is betty
  >
  > This is my dog, my dog's name is frank
  >
  > This is my monkey, my monkey's name is wukong
  >
  > This is my goat, my goat's name is adam

  

- > ##### p命令
  >
  > 打印命令
  >
  > 你可以把这个命令当成grep式的命令
  >
  > 
  >
  > \# 匹配fish并输出，可以看到fish的那一行被打了两遍，
  >
  > \# 这是因为sed处理时会把处理的信息输出
  >
  > $ sed '/fish/p' my.txt
  >
  > This is my cat, my cat's name is betty
  >
  > This is my dog, my dog's name is frank
  >
  > This is my fish, my fish's name is george
  >
  > This is my fish, my fish's name is george
  >
  > This is my goat, my goat's name is adam
  >
  > 
  >
  > \# 使用n参数就好了
  >
  > $ sed -n '/fish/p' my.txt
  >
  > This is my fish, my fish's name is george
  >
  > 
  >
  > \# 从一个模式到另一个模式
  >
  > $ sed -n '/dog/,/fish/p' my.txt
  >
  > This is my dog, my dog's name is frank
  >
  > This is my fish, my fish's name is george
  >
  > 
  >
  > \#从第一行打印到匹配fish成功的那一行
  >
  > $ sed -n '1,/fish/p' my.txt
  >
  > This is my cat, my cat's name is betty
  >
  > This is my dog, my dog's name is frank
  >
  > This is my fish, my fish's name is george
  >
  > 
  >
  > 匹配正确的ip地址
  >
  > ```shell
  > sed -n -r   "/((([0-9]{1,2})|(1[0-9]{2})|(2[0-4][0-9])|(25[0-5]))\.){3}/p" test_1 | awk -F "." '{if($4<255) print $0}'
  > ```

### awk

- sed的核心是**正则**，awk的核心是**格式化**.

  - 对于sed， 基本的两个概念是**匹配**和**行为**。
- awk它的运作模式是“**预处理+逐行处理+最终处理**”
    - 一般我们只用“逐行处理”比如对于满足条件的某些行，我们打印某某列。通过**指定分隔符，我们很容易的对列进行操作。**
    - **预处理来定义变量，逐行处理来修改变量，最终处理来打印变量。**
  
- awk是一个强大的文本分析工具，相对于grep的查找，sed的编辑，awk在其对数据分析并生成报告时，显得尤为强大。简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理

- > awk [选项参数] 'script' var=value file(s)
  >
  > awk [选项参数] -f scriptfile var=value file(s)

- 选项参数

  - > `awk -F`命令可以指定使用哪个分隔符，默认是空格或者 tab 键：

  > 

- > $ awk '{printf "%-8s %-8s %-8s %-18s %-22s %-15s\n",$1,$2,$3,$4,$5,$6}' netstat.txt
  >
  > 
  >
  > `$ awk -F ':' '{print NR ") " $1}' demo.txt`
  >
  > 每行按空格或TAB分割，输出文本中的1、4项
  >
  > $ awk '{print $1,$4}' log.
  >
  > 
  >
  > `awk '$2>=90{print $0}' score.txt`
  >
  > 
  >
  > 使用","分割
  >
  > $  awk -F, '{print $1,$2}'   log.txt
  >
  > 
  >
  > awk -v  # 设置变量
  >
  > $ awk -va=1 '{print $1,$1+a}' log.txt

#### 过滤记录

- $ awk '$3==0 && $6=="LISTEN" ' netstat.txt

- $ awk ' $3>0 {print $0}' netstat.txt

- 如果我们需要表头的话，我们可以引入内建变量NR：

  ​	$ awk '$3==0 && $6=="LISTEN" || NR==1 ' netstat.tx

#### **内建变量**

说到了内建变量，我们可以来看看awk的一些内建变量：

| $0       | 当前记录（这个变量中存放着整个行的内容）                     |
| -------- | ------------------------------------------------------------ |
| $1~$n    | 当前记录的第n个字段，字段间由FS分隔                          |
| FS       | 输入字段分隔符 默认是空格或Tab                               |
| NF       | 当前记录中的字段个数，就是有多少列                           |
| NR       | 已经读出的记录数，就是行号，从1开始，如果有多个文件话，这个值也是不断累加中。 |
| FNR      | 当前记录数，与NR不同的是，这个值会是各个文件自己的行号       |
| RS       | 输入的记录分隔符， 默认为换行符                              |
| OFS      | 输出字段分隔符， 默认也是空格                                |
| ORS      | 输出的记录分隔符，默认为换行符                               |
| FILENAME | 当前输入文件的名字                                           |

- 比如：我们如果要输出行号
  - $ awk '$3==0 && $6=="ESTABLISHED" || NR==1 {printf "%02s %s %-20s %-20s %s\n",NR, FNR, $4,$5,$6}' netstat.txt

#### 指定分隔符

> $  awk  'BEGIN{FS=":"} {print $1,$3,$6}' /etc/passwd
>
> 
>
> 上面的命令也等价于：（-F的意思就是指定分隔符）
>
> $ awk  -F: '{print $1,$3,$6}' /etc/passwd
>
> 
>
> 再来看一个以\t作为分隔符输出的例子（下面使用了/etc/passwd文件，这个文件是以:分隔的）：
>
> $ awk  -F: '{print $1,$3,$6}' OFS="\t" /etc/passwd

#### 字符串匹配

- > 我们再来看几个字符串匹配的示例：
  >
  > $ awk '$6 ~ /FIN/ || NR==1 {print NR,$4,$5,$6}' OFS="\t" netstat.txt
  >
  > 6       coolshell.cn:80 61.140.101.185:37538    FIN_WAIT2
  >
  > 9       coolshell.cn:80 116.234.127.77:11502    FIN_WAIT2
  >
  > 匹配FIN状态。其实 ~ 表示模式开始。/ /中是模式。这就是一个正则表达式的匹配。
  >
  > 我们可以使用 “/FIN|TIME/” 来匹配 FIN 或者 TIME
  >
  > 
  >
  > 模式取反  !~ /WAIT/

#### **折分文件**

- 在vi下面，跳到300行，输入:300回车

  如果不想打开，查看指定行，用sed：

  ```
  `sed` `-n ``'300p'` `file``.log`
  ```

  用awk：

  ```
  `awk` `'NR==300'` `file``.log`
  ```

- awk拆分文件很简单，使用重定向就好了。下面这个例子，是按第6例分隔文件，相当的简单（其中的NR!=1表示不处理表头）。

  > $ awk 'NR!=1{print > $6}' netstat.txt
  >
  > $ ls
  >
  > ESTABLISHED  FIN_WAIT1  FIN_WAIT2  LAST_ACK  LISTEN  netstat.txt  TIME_WAIT

- 你也可以把指定的列输出到文件：

  - awk 'NR!=1{print $4,$5 > $6}' netstat.txt

- 再复杂一点：（注意其中的if-else-if语句，可见awk其实是个脚本解释器）

  > $ awk 'NR!=1{if($6 ~ /TIME|ESTABLISHED/) print > "1.txt";
  >
  > else if($6 ~ /LISTEN/) print > "2.txt";
  >
  > else print > "3.txt" }' netstat.txt

#### 统计

- 下面的命令计算所有的C文件，CPP文件和H文件的文件大小总和。

  > $ ls -l  *.cpp *.c *.h | awk '{sum+=$5} END {print sum}'
  >
  > 2511401

- 我们再来看一个统计各个connection状态的用法：

  > $ awk 'NR!=1{a[$6]++;} END {for (i in a) print i ", " a[i];}' netstat.txt
  >
  > TIME_WAIT, 3
  >
  > FIN_WAIT1, 1
  >
  > ESTABLISHED, 6
  >
  > FIN_WAIT2, 3
  >
  > LAST_ACK, 1
  >
  > LISTEN, 4

- 再来看看统计每个用户的进程的占了多少内存（注：sum的RSS那一列）

  > $ ps aux | awk 'NR!=1{a[$1]+=$6;} END { for(i in a) print i ", " a[i]"KB";}' 
  >
  > dbus, 540KB 
  >
  > mysql, 99928KB 
  >
  > www, 3264924KB 
  
- 日志文件中访问量最大的top10 IP地址

  - > cat  test.log|awk -F" " '{print $2}'|sort|uniq -c|sort -nrk 1 -t' '|awk -F" " '{print $2}'|head -10
    >
    > sort|uniq -c   //统计重复的行数
    >
    > sort -n是按照数值进行由小到大进行排序， -r是表示逆序，-t是指定分割符，-k是执行按照第几列进行排序

#### awk脚本

- BEGIN{ 这里面放的是执行前的语句 }

- END {这里面放的是处理完所有的行后要执行的语句 }

- {这里面放的是处理每一行时要执行的语句}

- ```shell
  $ cat cal.awk
  #!/bin/awk -f
  #运行前
  BEGIN {
      math = 0
      english = 0
      computer = 0
  
      printf "NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL\n"
      printf "---------------------------------------------\n"
  }
  #运行中
  {
      math+=$3
      english+=$4
      computer+=$5
      printf "%-6s %-6s %4d %8d %8d %8d\n", $1, $2, $3,$4,$5, $3+$4+$5
  }
  #运行后
  END {
      printf "---------------------------------------------\n"
      printf "  TOTAL:%10d %8d %8d \n", math, english, computer
      printf "AVERAGE:%10.2f %8.2f %8.2f\n", math/NR, english/NR, computer/NR
  }
  ```

- 来看一下执行结果：（也可以这样运行 ./cal.awk score.txt）

  $ awk -f cal.awk score.txt

#### 环境变量

- （使用-v参数和ENVIRON，使用ENVIRON的环境变量需要export）

- ```shell
  $ x=5
  
  $ y=10
  $ export y
  
  $ echo $x $y
  5 10
  
  $ awk -v val=$x '{print $1, $2, $3, $4+val, $5+ENVIRON["y"]}' OFS="\t" score.txt
  Marry   2143    78      89      87
  Jack    2321    66      83      55
  Tom     2122    48      82      81
  Mike    2537    87      102     105
  Bob     2415    40      62      72
  ```

  

## 一般命令

- cp 命令，作用复制，参数如下：

  ```
  -a ：将文件的特性一起复制
  -p ：连同文件的属性一起复制，而非使用默认方式，与-a相似，常用于备份
  -i ：若目标文件已经存在时，在覆盖时会先询问操作的进行
  -r ：递归持续复制，用于目录的复制行为
  -u ：目标文件与源文件有差异时才会复制
  ```

-  mv命令作用为移动文件：

  ```
  -f ：force强制的意思，如果目标文件已经存在，不会询问而直接覆盖
  -i ：若目标文件已经存在，就会询问是否覆盖
  -u ：若目标文件已经存在，且比目标文件新，才会更新
  ```

- pwd命令，作用为查看”当前工作目录“的完整路径

  ```
  pwd -P # 显示出实际路径，而非使用连接（link）路径；pwd显示的是连接路径
  ```

- mkdir命令创建目录：

  ```
  mkdir [选项]... 目录... 
   -m, --mode=模式，设定权限<模式> (类似 chmod)，而不是 rwxrwxrwx 减 umask
   -p, --parents  可以是一个路径名称。此时若路径中的某些目录尚不存在,加上此选项后,系统将自动建立好那些尚不存在的目录,即一次可以建立多个目录; 
   -v, --verbose  每次创建新目录都显示信息
  ```

- .rmdir 命令删除目录：

  ```
  rmdir [选项]... 目录...
  -p 递归删除目录dirname，当子目录删除后其父目录为空时，也一同被删除。如果整个路径被删除或者由于某种原因保留部分路径，则系统在标准输出上显示相应的信息。 
  -v --verbose  显示指令执行过程 
  ```

-  ps 命令显示运行的进程，还会显示进程的一些信息如pid, cpu和内存使用情况等：

  ```
  -A ：所有的进程均显示出来
  -a ：不与terminal有关的所有进程
  -u ：有效用户的相关进程
  -x ：一般与a参数一起使用，可列出较完整的信息
  -l ：较长，较详细地将PID的信息列出
  ```

- . kill 命令用于终止进程，参数：

  ```
  kill -signal PID
  
  1：SIGHUP，启动被终止的进程
  2：SIGINT，相当于输入ctrl+c，中断一个程序的进行
  9：SIGKILL，强制中断一个进程的进行
  15：SIGTERM，以正常的结束进程方式来终止进程
  17：SIGSTOP，相当于输入ctrl+z，暂停一个进程的进行
  ```

- top 命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，类似于Windows的任务管理器：

  ```
  top [参数]
  -b 批处理
  -c 显示完整的治命令
  -I 忽略失效过程
  -s 保密模式
  -S 累积模式
  -i<时间> 设置间隔时间
  -u<用户名> 指定用户名
  -p<进程号> 指定进程
  ```

- ping 用于确定主机与外部连接状态：

  ```
  ping [参数] [主机名或IP地址]
  -d 使用Socket的SO_DEBUG功能。
  -f  极限检测。大量且快速地送网络封包给一台机器，看它的回应。
  -n 只输出数值。
  -q 不显示任何传送封包的信息，只显示最后的结果。
  -r 忽略普通的Routing Table，直接将数据包送到远端主机上。通常是查看本机的网络接口是否有问题。
  -R 记录路由过程。
  -v 详细显示指令的执行过程。
  <p>-c 数目：在发送指定数目的包后停止。
  -i 秒数：设定间隔几秒送一个网络封包给一台机器，预设值是一秒送一次。
  -I 网络界面：使用指定的网络界面送出数据包。
  -l 前置载入：设置在送出要求信息之前，先行发出的数据包。
  -p 范本样式：设置填满数据包的范本样式。
  -s 字节数：指定发送的数据字节数，预设值是56，加上8字节的ICMP头，一共是64ICMP数据字节。
  -t 存活数值：设置存活数值TTL的大小。
  ```

- ifconfig　命令用来查看和配置网络设备。当网络环境发生改变时可通过此命令对网络进行相应的配置

  ```
  -a 显示全部接口信息
  -s 显示摘要信息（类似于 netstat -i）
  ```

- ln 命令是为某一个文件在另外一个位置建立一个同步的链接

  ```
  Linux文件系统中，有所谓的链接(link)，我们可以将其视为档案的别名，而链接又可分为两种 : 硬链接(hard link)与软链接(symbolic link)，硬链接的意思是一个档案可以有多个名称，而软链接的方式则是产生一个特殊的档案，该档案的内容是指向另一个档案的位置。硬链接是存在同一个文件系统中，而软链接却可以跨越不同的文件系统。
  
  软链接：
  1.软链接，以路径的形式存在。类似于Windows操作系统中的快捷方式
  2.软链接可以 跨文件系统 ，硬链接不可以
  3.软链接可以对一个不存在的文件名进行链接
  4.软链接可以对目录进行链接
  
  硬链接:
  1.硬链接，以文件副本的形式存在。但不占用实际空间。
  2.不允许给目录创建硬链接
  3.硬链接只有在同一个文件系统中才能创建
  
  ln [参数][源文件或目录][目标文件或目录]
  
  必要参数:
  -b 删除，覆盖以前建立的链接
  -d 允许超级用户制作目录的硬链接
  -f 强制执行
  -i 交互模式，文件存在则提示用户是否覆盖
  -n 把符号链接视为一般目录
  -s 软链接(符号链接)
  -v 显示详细的处理过程
  
  选择参数:
  -S “-S<字尾备份字符串> ”或 “--suffix=<字尾备份字符串>”
  -V “-V<备份方式>”或“--version-control=<备份方式>”
  ```

  

-  ls 命令，展示文件夹内内容，参数如下：

  ```
  -a ：全部的档案，连同隐藏档( 开头为 . 的档案) 一起列出来～ 
  -A ：全部的档案，连同隐藏档，但不包括 . 与 .. 这两个目录，一起列出来～ 
  -d ：仅列出目录本身，而不是列出目录内的档案数据 
  -f ：直接列出结果，而不进行排序 (ls 预设会以档名排序！) 
  -F ：根据档案、目录等信息，给予附加数据结构，例如： 
  *：代表可执行档； /：代表目录； =：代表 socket 档案； |：代表 FIFO 档案； 
  -h ：将档案容量以人类较易读的方式(例如 GB, KB 等等)列出来； 
  -i ：列出 inode 位置，而非列出档案属性； 
  -l ：长数据串行出，包含档案的属性等等数据； 
  -n ：列出 UID 与 GID 而非使用者与群组的名称 (UID与GID会在账号管理提到！) 
  -r ：将排序结果反向输出，例如：原本档名由小到大，反向则为由大到小； 
  -R ：连同子目录内容一起列出来； 
  -S ：以档案容量大小排序！ 
  -t ：依时间排序 
  --color=never ：不要依据档案特性给予颜色显示； 
  --color=always ：显示颜色 
  --color=auto ：让系统自行依据设定来判断是否给予颜色 
  --full-time ：以完整时间模式 (包含年、月、日、时、分) 输出 
  --time={atime,ctime} ：输出 access 时间或 改变权限属性时间 (ctime) 
  而非内容变更时间 (modification time)  
     
  例如：
  ls [-aAdfFhilRS] 目录名称 
  ls [--color={none,auto,always}] 目录名称 
  ls [--full-time] 目录名称  
  ```

- chown

  - > chown -R zzh:sk /app/oracle 
    >  更改目录拥有者为zzh

- chmod

  - 修改权限

  - > chmod {u|g|o|a}{+|-|=}{r|w|x} filename

  - > chmod abc file 
    >
    >   若要rwx属性则4+2+1=7； 
    >   若要rw-属性则4+2=6； 
    >   若要r-x属性则4+1=5。 
    > 	      4 (100)    表示可读。 
    >           2 (010)    表示可写。 
    >           1 (001)    表示可执行。 
    >
    >  其中a,b,c各为一个八进制数字，分别表示User、Group、及Other的权限。 
    > chmod 741 filename 
    >
    > ​        让本人可读写执行、同组用户可读、其他用户可执行文件filename。 
    >
    > chmod -R 755 /home/oracle 
    >
    > 递归更改目录权限，本人可读写执行、同组用户可读可执行、其他用户可读可执行 

- head 命令用于显示档案的开头至标准输出中，默认head命令打印其相应文件的开头10行

  - head -n 20 /www/server/php/72/etc/php.ini 显示前20条
  
- tail

  - >  -i     显示文件最后 i行。 
    >  +i    从文件的第i行开始显示。
    >  tail -n 1000：显示最后1000行
    >  tail -n +1000：从1000行开始显示，显示1000行以后的
    >  head -n 1000：显示前面1000行     
    >
    >  tail -f a.log 可以查看文件最后增加的内容
    
  - 

  ```shell
  1.head命令查看文件中的前200行
  
  head -n 200 filename
  2.tail 命令查看文件中的后100行
  
  tail -n 100 filename
  3.查看文件100行到200行
  
  head -n 200 filename | tail -n 100
  4.从100行开始显示文件
  
  tail -n +100 filename
   
  5.显示除后100行的文件内容
  
  head -n -100 filename
  ```

- find

  - 在所给的路经名下寻找符合表达式相匹配的文件。 

  - > find pathname [option] expression 
    >
    > -name     表示文件名 
    >
    >  -type     按文件类型查找 

- more&less

  - more&less最重要的一点就是流式读取，支持翻页，像cat命令是全部读取输出到标准输出，如果文件太大会把屏幕刷满的，根本没办法看。
  - less 与 more 类似，但使用 less 可以随意浏览文件，而 more 仅能向前移动，却不能向后移动，而且 less 在查看之前不会加载整个文件
  - less：配合 [pageup] [pagedown] 等按键的功能来往前往后翻看文件

- wc

  - > -c 统计字节数。
    >
    > -l 统计行数。
    >
    > -m 统计字符数。这个标志不能与 -c 标志一起使用。
    >
    > -w 统计字数。一个字被定义为由空白、跳格或换行字符分隔的字符串。
    >
    > -L 打印最长行的长度。

- cut

  - > $cut -c-10 tmp.txt  #cut tmp.txt文件的前10列
    > $cut -c3-5 tmp.txt  #cut tmp.txt文件的第3到5列
    > $cut -c3- tmp.txt  #cut tmp.txt文件的第3到结尾列

- 查看占用端口的进程

  使用netstat，示例：查看特定端口3366的进程

  ```bash
  # netstat -anp | grep 3366
  ```

  使用lsof，lsof -i:端口号查看某个端口是否被占用

  ```bash
  # lsof -i:3366
  ```

  































