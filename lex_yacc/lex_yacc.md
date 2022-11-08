# LEX and YACC
## lex notes
lex包含三部分，`Definition Section`, `Rules Section`, `User Subroutine Section`
*******
### Definition Section
```lex
%{
    /*
    any c code
    */
    #include <string.h>
    //include header files
%}
word [^ \t\n]+
eol \n
%%
```
`%%`意思是C代码包裹的结束符
******
### Rules Section
每一个`rule`包含一个`pattern`和一个`action`, 中间是空格  
#### Pattern
pattern其实就是unix-style正则表达式以及grep,sed,ed使用的正则扩展。  
没有匹配的text会被默认输出到`yyout`, 也就是`ECHO`.  
如果你想确保自己匹配了所有可能性，那么`flex`可以指定`-s`选项用于把这一默认行为改为abort.  
```lex
%{
    /*
    */
%}
%%
[\t ]+ /*ignore whitespace*/ ;

foo|bar { printf("foo or bar will go here");}

do |
does |
did |
go { printf("%s: is a verb\n", yytext);}
[a-zA-Z]+ { printf("%s: is not verb\n", yytext);}
.|\n { ECHO;/*default way*/}
%%
```
`pattern`: `[\t ]+`  
`action`: `;`
#### Action 其实就是C代码
`|` 是一个特殊的`action`，表示具体action与下一个action相同。  
最后的`%%`是结束符。
******
### User Subroutine Section
C代码, lex会把这些内容拷贝到输出。
```lex
%%
main() {
    yylex();
}
```
其实我们可以不写main函数，因为lex输出会默认带一个main函数。  
除非某个`pattern`的`action`有`return`操作，否则`yylex`会在处理完所有输入之后才会返回.

*******
### 关于Context Sensitivity
是指在lexer解析的时候能够根据前后的多余信息进行更精确的匹配.
#### left context
有三种方式来处理左上下文:  
- 正则中的`^`, 正则中`^`只能用于`pattern`的开头  
- start states, like `BEGIN STATE1`, `BEGIN INITIAL`
- 用代码

#### right context
- 正则的 `$`
- 正则的 `/`
- `yless()`

正则 `/` 有个需要注意的点:  
如: 
```lex
%%
abc/de {printf("%s, %d\n", yytext, yyleng); }
%%
```
只有在遇到`abcde`的时候, 才会匹配到这个规则，但是打印的内容是`abc，3`, 而不是`abcde, 5`  

`yyless`在匹配到一个pattern之后，可以用来放回去一些字符.  
参数是指保留多少个字符， 因此yyless(3)的意思就是保留三个.  
```lex
abcde {yyless(3); printf("%s, %d\n", yytext, yyleng);}
```
所以上面的意思和`abc/de`是一样的，唯一的区别就是`yytext`是`abcde`， `yyleng`是5
`yyless`的一个例子:  
```lex
\" [^"]*\"    {
                if (yytext[yyleng - 2] == '\\')a {
                  yyless(yyleng - 1);
                  yymore();
                } else {
                  //...
                }
              }
```
这个是为了匹配多行字符串的场景,如:  
```c
"asdasdasd\
asdasdasd"
```
如果第一行解析到了`\`, 那么将这个反斜杠放回去, 然后`yymore()`.  
`yymore`是为了告诉lex把下一个token的内容和当前这个当成一个token.  
当读完之后 返回整个字符串`asdasdasd\nasdasdasd`, 带换行符的.  
*********
### lex单词计数的例子
```lex
%{
    /*
    */
unsigned charCount = 0, wordCount = 0, lineCount = 0;
%}
%}
word [^ \t\n]+
eol \n
%%

{word} { wordCount++; charCount += yyleng; }
{eol} { charCount++; lineCount++; }
. charCount++;
%%

main(argc,argv)
int argc;
char **argv;
{
    if (argc > 1) {
        FILE *file;
        file = fopen(argv[1], "r");
        if (!file) {
            fprintf(stderr,"could not open %s\n",argv[1]);
            exit(1);
        }
        yyin = file;
    }
    yylex();
    printf("%u %u %u\n",charCount, wordCount, lineCount);
    return 0;
}
```
`Rule Section`内的`{word}`会被替换成`Definition Section`部分word对应的正则。  
```lex
%{
%}
PAT abc
%%
{PAT}+ ;
```
这个在替换的可能会替换为`abc+`, 于是就成了最后一个字符重复多遍  
如果是期望(abc)+, 那么就可能需要将PAT定义成: `PAT (abc)`.  

`yyleng`是当前匹配word的长度。  
首先匹配先出现的规则最后匹配`.`这个规则.  
`yyin`就是`yylex`的输入，默认是从`stdin`  
`yywrap`用来切换`yyin`的，当`yyin`读到文件结尾时, `yylex`会调用`yywrap`,如果`yywrap`返回1那么表示结束，0表示可以接着读，并且期望在`yywrap`里已经重新打开了新文件, `yyin`已经设置好了。  
要重写`yywrap`, `input`, `unput`等函数需要先undef  
```lex
%{
#undef yywrap
#undef input
#undef unput
int yywrap();
int input(void);
void unput(int ch);
%}
%%
```
### lex用来做命令行参数解析的列子
```lex
%{
#undef input
#undef unput
unsigned verbose
unsigned fname;
char *progName;
%}

%s FNAME
%x CMNT
%%
"/*"        BEGIN CMNT; /*begin comment*/
<CMNT>"*/"  BEGIN INITIAL;/*return to initial state*/

[ ]+ /* ignore blanks */ ;
-h |
"-?" |
-help { printf("usage is: %s [-help | -h | -? ] [-verbose | -v]"
    " (-file | -f) filename\n", progName);
}
-v |
-verbose { printf("verbose mode is on\n"); verbose = 1; }
-f |
-file { BEGIN FNAME; fname = 1; }
<FNAME>[^ ]+ { printf("use file %s\n", yytext); BEGIN 0; fname = 2;}
[^ ] + ECHO;
%%

char **targv; /* remembers arguments */
char **arglim; /* end of arguments */
main(int argc, char **argv)
{
    progName = *argv;
    targv = argv+1;
    arglim = argv+argc;
    yylex();
    if(fname < 2)
    printf("No filename given\n");
}

static unsigned offset = 0;
int input(void)
{
    char c;
    if (targv >= arglim) return(0); /* EOF */
    /* end of argument, move to the next */
    if ((c = targv[0][offset++]) != '\0') return(c);
    targv++;
    offset = 0;
    return(' ');
}
/* simple unput only backs up, doesn't allow you to */
/* put back different text */
void
unput(int ch)
{
    /* AT&T lex sometimes puts back the EOF ! */
    if(ch == 0) return; /* ignore, can't put back EOF */
    if (offset) { /* back up in current arg */
        offset−−;
        return;
    }
    targv−−; /* back to previous arg */
    offset = strlen(*targv);
}
```
`input`, `unput`在`AT&T` lex支持重新define，貌似`flex`不支持  
这两个函数在`AT&T` lex中作为宏存在，我们可以将它重新定义为函数。  

`yylex`会反复调用`input`获取字符  
`input`返回0表示`EOF`, 返回空格表示下一个token, 否则返回一个字符.  
`unput`放回一个字符,其实就是把当前指针回退一格, 具体什么时候被调用没看懂...  

`%s FNAME`定义了一个`start state`  
用来标记一些具有上下文信息的规则, 如`<FNAME> [^ ]+ {printf("use file %s\n", yytext); BEGIN 0; fname = 2;}`  
`%x`也是定义了一个`start state`, x和s的区别是:  
如果已经`BEGIN`了`CMNT`, 那么只会匹配当前状态下的规则.  
```lex
%{
%}
%s STATE1
%x STATE2
%%
OK          {BEGIN INITIAL; printf("unknow state ok\n");}
<STATE1>OK  {BEGIN INITIAL; printf("STATE1 OK\n");}
<STATE2>OK  {BEGIN INITIAL; printf("STATE2 OK\n");}
```
在进入`STATE1`之后, 如果解析到OK会匹配第一个OK,打印`unknow state ok`.  
如果进入`STATE2`之后, 如果解析到OK不会匹配第一个OK, 而是匹配第三个OK, 打印`STATE2 OK`  

上面匹配到`-file {BEGIN FNAME; fname = 1;}`时就会开启该`start state`  
标记之后lexer只会在当前状态处于`FNAME`时才会匹配到此规则.  
于是文件的名字就可以单独被解析出来, 此规则的`action`部分会`BEGIN 0`, `0`或者`INITIAL`, 重置状态.  



### 编译

```bash
lex 1.l
#生成lex.yy.c
yacc -d 1.y
#生成y.tab.c y.tab.h
#y.tab.h包含token定义使用-d选项
cc -c lex.yy.c y.tab.c
cc -o a.out lex.yy.o y.tab.o -ll
```
*******
## yacc notes
yacc 包括下面几部分
### Definition Section
与lex相同, 但是多了个`TOKEN`的定义部分
```yacc
%{
#include <stdio.h>
%}
%token NOUM PRONOUM VERB
%start stmt

%left '+' '-'
%left '*' '/'
%nonassoc UMINUS

%%
```
一般情况下`TOKEN`最好都是大写，其他用小写
`%start`是指定`start symbol`, 不指定的话就是所有规则中的第一个规则.
`%nonassoc`, `%left`, `%right`用于设定`assiciativity`, 后面会提.

*******
### Rules Section
```yacc
sentence: subject VERB object {printf("Sentence is valid.\n");}
    ;
subject:    NOUN
    |       PRONOUN
    ;
object:     NOUM
    ;
%%
```

*******
### User Subroutines Section
```yacc
extern FILE *yyin;
main()
{
    do {
        yyparse();
    } while (!feof(yyin));
}
yyerror(s)
char *s;
{
    fprintf(stderr, "%s\n", s);
}
```
lexer在返回0时表示当前语句解析结束。
*******
### shift and reduction
```yacc
statement: number
  | expression + number
  | expression - number
```
`statement`就是左边部分.
其他剩下的就是右边部分.
parser在读取token时，若读完当前token不能满足某一个规则，则把当前token入栈  
切换到新读取的token的状态, 这个切换过程就叫做shift。  
当parser读取token之后，满足了某一个语法规则，那么parser将该规则的右边所有的符号都pop出栈，并把该规则的左边入栈, 该过程叫做reduction。  

*******
### What yacc can't parse?
yacc 不能解析歧义语法, 也不能在解析的时候向前peek 2个token, 最多一个.
如:
```yacc
phrase:   cart_animal AND CART
        | work_animal AND PLOW

cart_animal:  HORSE | GOAT
work_animal:  HORSE | OX
```
上面的语法没有歧义,但是如果解析到`HORSE`, yacc不知道是`cart_animal`的还是`work_animal`的`HORSE`, 除非向前再解析两个token, 如果是`AND CART`那个就是`cart_animal`, 如果是`AND PLOW`那么就是`work_animal`.  
但是yacc不支持.   

********
### Assiciativity and Precedence
问题:
```yacc
expression: expression '+' expression {$$ = $1 + $2;}
        |   expression '-' expression {$$ = $1 - $2;}
        |   expression '*' expression {$$ = $1 * $2;}
        |   expression '/' expression {$$ = $1 / $2;/*divide zero handling*/}
        |   '-' expression {$$ = -$2;}
        |   '(' expression ')' {$$ = $2;}
        |   NUMBER {$$ = $1;}
        ;
```
我们需要运算符的优先级, 以上不能提供匹配的优先级, 默认就是从左到右的匹配.  
因此`1+2*3`返回的结果是9, 不是7.  
于是就有了`Precedence`和`Associativity`  
`Precedence`用于控制语法匹配的优先权.  
`Associativity`用于控制在相同`precedence level`的语法解析时如何组合.  
```yacc
%{
%left '+' '-'
%left '*' '/'
%nonassoc UMINUS
%}
%%

expression: expression '+' expression {$$ = $1 + $2;}
        |   expression '-' expression {$$ = $1 - $2;}
        |   expression '*' expression {$$ = $1 * $2;}
        |   expression '/' expression {$$ = $1 / $2;/*divide zero handling*/}
        |   '-' expression %prec UMINUS {$$ = -$2;}
        |   '(' expression ')' {$$ = $2;}
        |   NUMBER {$$ = $1;}
        ;
%%
```
`%left`意思是`'+', '-', '*', '/'`都是左关联的运算符, `left associative`.  
`%nonassoc`意思是不关联  
`UMIUNS`就是unary minus, 一元运算符减号.  
从上倒下`'+','-'`的`precedence level`最低, 然后是`'*', '/'`, 最高的是`'UMIUS'`  
于是: `1+2*-3` 就会这样解析  
|解析stack|操作|
|:-----|:-----|
|NUMBER |                                              接着看到'+', 没有任何相关的规则,直接reduce成|
|expression |                                          接着把'+'也shift进去得到:|
|expression '+' |                                      接着读到'2', 把NUMBER shift 进去,|
|expression '+' NUMBER |                               reduce成:|
|expression '+' expression |                           发现匹配到了加号的规则, 但是下一个字符是'*', `precedence level`较高, 于是先不做`reduce`, 而是接着做shift|
|expression '+' expression '*'|                         接着读到'-', 没有匹配到相关规则, 于是shift|
|expression '+' expression '*' '-'|                     接着读到'3', shift|
|expression '+' expression '*' '-' NUMBER|              接着读, 没发现后面有更高`precendence level`的字符了, 于是开始匹配, 匹配到了负号的规则, 开始做reduce|
|expression '+' expression '*' expression|              接着匹配，发现了乘法规则，于是reduce|
|expression '+' expression|                            接着reduce|
|expression|                                           结束

`yacc`会将规则的最右边的`token`设置为该规则的`precedence level`.  
如果该规则没有配置`precendece level`的token,那么该规则就没有`precedence level`.  
当`yacc`遇到`shift/reduce`冲突时, 就是yacc不知道该`reduce`还是`shift`, yacc会去查询`precedences表`.  

我们已经定义了`'-'`的`precedence level`, 为最低, `UMINUS`最高.  
`%prec UMINUS`为了将这个负号规则使用`UMINUS`的`precendece level`, 而不是减号的.  
******

### yacc的计算器的例子(见cpp/lex_yacc/calculator.y 和calculator.l)
```yacc
%union {
  double dval;
  int vblno;
}
%token <vblno> NAME
%token <dval> NUMBER
%left '-' '+'
%left '*' '/'
%nonassoc UMINUS
%type <dval> expression
%%
```
里面定义个一个union, 这个union会在编译时生成一个C的union:
```c
/* Tokens.  */
#define NAME 258
#define NUMBER 259
#define UMINUS 260
/* Value type.  */
#if ! defined YYSTYPE && ! defined YYSTYPE_IS_DECLARED
union YYSTYPE
{
#line 6 "2.y" /* yacc.c:1909  */
  double dval;
  int vblno;
#line 69 "y.tab.h" /* yacc.c:1909  */
};
typedef union YYSTYPE YYSTYPE;
# define YYSTYPE_IS_TRIVIAL 1
# define YYSTYPE_IS_DECLARED 1
#endif
```
这个`YYSTYPE`是一个总的类型定义  
每个token, 每个规则的左边都可以定义类型, 如`NAME`类型是字符串数组中的下标,也就是int, `NUMBER`是double value, 如`%left '-'`也可以定义(不过这儿不需要),`expression`的类型定义成了`double value`.  
给`NUMBER`或者`NAME`定义了类型之后, 如这个语法规则:
```yacc
expression: expression '+' expression {/*...*/}
  | NAME {$$ = vbltable[$1];}
```
当匹配到NAME之后, 在`取NAME的$1`的时候yacc才知道将`$1`替换为`$1.vblno`;  
并且`$$`会替换为`$$.dval`, 因为`expression`是一个`double`类型.  

******
### Embedded Actions in yacc
```yacc
%%
thing: A {printf("seen an A");} B;
%%
```
yacc会把它解析为:  
```yacc
%%
thing: A fakename B;
fakename: /*empty*/ {printf("seen an A");}
%%
```
于是`$2`就是这个`fakename`的返回值, 虽然这儿没有返回值.  

yacc的这种替换也会有问题:
```yacc
%%
thing: abcd | abcz;

abcd: 'A' 'B' {somefunc();} 'C' 'D';
abcz: 'A' 'B' 'C' 'Z';
```
当yacc解析完B之后, 如果没有这个`{somefunc();}`, 那么yacc就可以不用现在决定自己的匹配的到底是`abcd`还是`abcz`,  
但是有了这个之后, yacc必须现在决定, 但是由于读到后面的`C`还是不能确定,于是就处理不了了,  
如果在C的后面, 那么yacc是可以处理的.  

如果你要引用这个`embedded action`的返回值, 并且使用`%union`定义了类型,  
那么你在引用的时候就需要指定类型:    
- `$$` -> `$<type>$`
- `$3` -> `$<type>3`

************
