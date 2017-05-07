# PEG

[NAME](#NAME)  
[SYNOPSIS](#SYNOPSIS)  
[DESCRIPTION](#DESCRIPTION)  
[OPTIONS](#OPTIONS)  
[A SIMPLE EXAMPLE](#SIMPLE_EXAMPLE)  
[PEG GRAMMARS](#PEG_GRAMMARS)  
[PEG GRAMMAR FOR PEG GRAMMARS](#PEG_GRAMMAR_FOR_PEG_GRAMMARS)  
[LEG GRAMMARS](#LEG_GRAMMARS)  
[LEG EXAMPLE: A DESK CALCULATOR](#LEG_EXAMPLE_A_DESK_CALCULATOR)  
[LEG GRAMMAR FOR LEG GRAMMARS](#LEG_GRAMMAR_FOR_LEG_GRAMMARS)  
[CUSTOMISING THE PARSER](#CUSTOMISING_THE_PARSER)  
[LEG EXAMPLE: EXTENDING THE PARSER’S CONTEXT](#LEG_EXAMPLE_EXTENDING_THE_PARSERS_CONTEXT)  
[DIAGNOSTICS](#DIAGNOSTICS)  
[CAVEATS](#CAVEATS)  
[BUGS](#BUGS)  
[SEE ALSO](#SEE_ALSO)  
[AUTHOR](#AUTHOR)  

* * *

## NAME<a name="NAME"></a>

peg, leg − parser generators

## SYNOPSIS<a name="SYNOPSIS"></a>

**peg [−hvV −ooutput]** _[filename ...]_

**leg [−hvV −ooutput]** _[filename ...]_

## DESCRIPTION<a name="DESCRIPTION"></a>

_peg_ and _leg_ are tools for generating recursive−descent parsers: programs that perform pattern matching on text. They process a Parsing Expression Grammar (PEG) [Ford 2004] to produce a program that recognises legal sentences of that grammar. _peg_ processes PEGs written using the original syntax described by Ford; _leg_ processes PEGs written using slightly different syntax and conventions that are intended to make it an attractive replacement for parsers built with _lex_(1) and _yacc_(1). Unlike _lex_ and _yacc_, _peg_ and _leg_ support unlimited backtracking, provide ordered choice as a means for disambiguation, and can combine scanning (lexical analysis) and parsing (syntactic analysis) into a single activity.

_peg_ reads the specified _filename_s, or standard input if no _filename_s are given, for a grammar describing the parser to generate. _peg_ then generates a C source file that defines a function _yyparse()._ This C source file can be included in, or compiled and then linked with, a client program. Each time the client program calls _yyparse_() the parser consumes input text according to the parsing rules, starting from the first rule in the grammar. _yyparse_() returns non−zero if the input could be parsed according to the grammar; it returns zero if the input could not be parsed.

The prefix ’yy’ or ’YY’ is prepended to all externally−visible symbols in the generated parser. This is intended to reduce the risk of namespace pollution in client programs. (The choice of ’yy’ is historical; see _lex_(1) and _yacc_(1), for example.)

## OPTIONS<a name="OPTIONS"></a>

_peg_ and _leg_ provide the following options:

<table rules="none" frame="void" border="0" cellpadding="0" cellspacing="0" width="100%">

<tbody>

<tr align="left" valign="top">

<td width="3%">

**−h**

</td>

<td width="78%">

prints a summary of available options and then exits.

</td>

</tr>

</tbody>

</table>

**−ooutput**

writes the generated parser to the file **output** instead of the standard output.

<table rules="none" frame="void" border="0" cellpadding="0" cellspacing="0" width="100%">

<tbody>

<tr align="left" valign="top">

<td width="3%">

**−P**

</td>

<td width="78%">

suppresses #line directives in the output.

</td>

</tr>

<tr align="left" valign="top">

<td width="3%">

**−v**

</td>

<td width="78%">

writes verbose information to standard error while working.

</td>

</tr>

<tr align="left" valign="top">

<td width="3%">

**−V**

</td>

<td width="78%">

writes version information to standard error then exits.

</td>

</tr>

</tbody>

</table>

## A SIMPLE EXAMPLE<a name="SIMPLE_EXAMPLE"></a>

The following _peg_ input specifies a grammar with a single rule (called ’start’) that is satisfied when the input contains the string "username".

start <− "username"

(The quotation marks are _not_ part of the matched text; they serve to indicate a literal string to be matched.) In other words, _yyparse_() in the generated C source will return non−zero only if the next eight characters read from the input spell the word "username". If the input contains anything else, _yyparse_() returns zero and no input will have been consumed. (Subsequent calls to _yyparse_() will also return zero, since the parser is effectively blocked looking for the string "username".) To ensure progress we can add an alternative clause to the ’start’ rule that will match any single character if "username" is not found.

start <− "username"  
/ .

_yyparse_() now always returns non−zero (except at the very end of the input). To do something useful we can add actions to the rules. These actions are performed after a complete match is found (starting from the first rule) and are chosen according to the ’path’ taken through the grammar to match the input. (Linguists would call this path a ’phrase marker’.)

start <− "username" { printf("%s\n", getlogin()); }  
/ < . > { putchar(yytext[0]); }

The first line instructs the parser to print the user’s login name whenever it sees "username" in the input. If that match fails, the second line tells the parser to echo the next character on the input the standard output. Our parser is now performing useful work: it will copy the input to the output, replacing all occurrences of "username" with the user’s account name.

Note the angle brackets (’<’ and ’>’) that were added to the second alternative. These have no effect on the meaning of the rule, but serve to delimit the text made available to the following action in the variable _yytext_.

If the above grammar is placed in the file **username.peg**, running the command

peg −o username.c username.peg

will save the corresponding parser in the file **username.c**. To create a complete program this parser could be included by a C program as follows.
```c
#include <stdio.h> /* printf(), putchar() */  
#include <unistd.h> /* getlogin() */

#include "username.c" /* yyparse() */

int main()  
{  
while (yyparse()) /* repeat until EOF */  
;  
return 0;  
}
```
## PEG GRAMMARS<a name="PEG_GRAMMARS"></a>

A grammar consists of a set of named rules.

name <− pattern

The **pattern** contains one or more of the following elements.

<table rules="none" frame="void" border="0" cellpadding="0" cellspacing="0" width="100%">

<tbody>

<tr align="left" valign="top">

<td width="6%">

**name**

</td>

<td width="78%">

The element stands for the entire pattern in the rule with the given **name**.

</td>

</tr>

</tbody>

</table>

**"**characters**"**

A character or string enclosed in double quotes is matched literally. The ANSI C escape sequences are recognised within the _characters_.

**’**characters**’**

A character or string enclosed in single quotes is matched literally, as above.

**[**characters**]**

A set of characters enclosed in square brackets matches any single character from the set, with escape characters recognised as above. If the set begins with an uparrow (^) then the set is negated (the element matches any character _not_ in the set). Any pair of characters separated with a dash (−) represents the range of characters from the first to the second, inclusive. A single alphabetic character or underscore is matched by the following set.

[a−zA−Z_]

Similarly, the following matches any single non−digit character.

[^0−9]

<table rules="none" frame="void" border="0" cellpadding="0" cellspacing="0" width="100%">

<tbody>

<tr align="left" valign="top">

<td width="1%">

**.**

</td>

<td width="78%">

A dot matches any character. Note that the only time this fails is at the end of file, where there is no character to match.

</td>

</tr>

</tbody>

</table>

**( **pattern **)**

Parentheses are used for grouping (modifying the precedence of the operators described below).

**{ **action **}**

Curly braces surround actions. The action is arbitrary C source code to be executed at the end of matching. Any braces within the action must be properly nested. Any input text that was matched before the action and delimited by angle brackets (see below) is made available within the action as the contents of the character array _yytext_. The length of (number of characters in) _yytext_ is available in the variable _yyleng_. (These variable names are historical; see _lex_(1).)

<table rules="none" frame="void" border="0" cellpadding="0" cellspacing="0" width="100%">

<tbody>

<tr align="left" valign="top">

<td width="1%">

**<**

</td>

<td width="78%">

An opening angle bracket always matches (consuming no input) and causes the parser to begin accumulating matched text. This text will be made available to actions in the variable _yytext_.

</td>

</tr>

<tr align="left" valign="top">

<td width="1%">

**>**

</td>

<td width="78%">

A closing angle bracket always matches (consuming no input) and causes the parser to stop accumulating text for _yytext_.

</td>

</tr>

</tbody>

</table>

The above _element_s can be made optional and/or repeatable with the following suffixes:  
element **?**

The element is optional. If present on the input, it is consumed and the match succeeds. If not present on the input, no text is consumed and the match succeeds anyway.

element **+**

The element is repeatable. If present on the input, one or more occurrences of _element_ are consumed and the match succeeds. If no occurrences of _element_ are present on the input, the match fails.

element *****

The element is optional and repeatable. If present on the input, one or more occurrences of _element_ are consumed and the match succeeds. If no occurrences of _element_ are present on the input, the match succeeds anyway.

The above elements and suffixes can be converted into predicates (that match arbitrary input text and subsequently succeed or fail _without_ consuming that input) with the following prefixes:**  
& **element

The predicate succeeds only if _element_ can be matched. Input text scanned while matching _element_ is not consumed from the input and remains available for subsequent matching.

**! **element

The predicate succeeds only if _element_ cannot be matched. Input text scanned while matching _element_ is not consumed from the input and remains available for subsequent matching. A popular idiom is

!.

which matches the end of file, after the last character of the input has already been consumed.

A special form of the ’&’ predicate is provided:**  
&**{ expression }

In this predicate the simple C _expression_ (**not** statement) is evaluated immediately when the parser reaches the predicate. If the _expression_ yields non−zero (true) the ’match’ succeeds and the parser continues with the next element in the pattern. If the _expression_ yields zero (false) the ’match’ fails and the parser backs up to look for an alternative parse of the input.

Several elements (with or without prefixes and suffixes) can be combined into a _sequence_ by writing them one after the other. The entire sequence matches only if each individual element within it matches, from left to right.

Sequences can be separated into disjoint alternatives by the alternation operator ’/’.  
sequence−1 **/ **sequence−2 **/ **... **/ **sequence−N

Each sequence is tried in turn until one of them matches, at which time matching for the overall pattern succeeds. If none of the sequences matches then the match of the overall pattern fails.

Finally, the pound sign (#) introduces a comment (discarded) that continues until the end of the line.

To summarise the above, the parser tries to match the input text against a pattern containing literals, names (representing other rules), and various operators (written as prefixes, suffixes, juxtaposition for sequencing and and infix alternation operator) that modify how the elements within the pattern are matched. Matches are made from left to right, ’descending’ into named sub−rules as they are encountered. If the matching process fails, the parser ’back tracks’ (’rewinding’ the input appropriately in the process) to find the nearest alternative ’path’ through the grammar. In other words the parser performs a depth−first, left−to−right search for the first successfully−matching path through the rules. If found, the actions along the successful path are executed (in the order they were encountered).

Note that predicates are evaluated _immediately_ during the search for a successful match, since they contribute to the success or failure of the search. Actions, however, are evaluated only after a successful match has been found.

## PEG GRAMMAR FOR PEG GRAMMARS<a name="PEG_GRAMMAR_FOR_PEG_GRAMMARS"></a>

The grammar for _peg_ grammars is shown below. This will both illustrate and formalise the above description.

Grammar <− Spacing Definition+ EndOfFile

Definition <− Identifier LEFTARROW Expression  
Expression <− Sequence ( SLASH Sequence )*  
Sequence <− Prefix*  
Prefix <− AND Action  
/ ( AND | NOT )? Suffix  
Suffix <− Primary ( QUERY / STAR / PLUS )?  
Primary <− Identifier !LEFTARROW  
/ OPEN Expression CLOSE  
/ Literal  
/ Class  
/ DOT  
/ Action  
/ BEGIN  
/ END

Identifier <− < IdentStart IdentCont* > Spacing  
IdentStart <− [a−zA−Z_]  
IdentCont <− IdentStart / [0−9]  
Literal <− [’] < ( ![’] Char )* > [’] Spacing  
/ ["] < ( !["] Char )* > ["] Spacing  
Class <− ’[’ < ( !’]’ Range )* > ’]’ Spacing  
Range <− Char ’−’ Char / Char  
Char <− ’\\’ [abefnrtv’"\[\]\\]  
/ ’\\’ [0−3][0−7][0−7]  
/ ’\\’ [0−7][0−7]?  
/ ’\\’ ’−’  
/ !’\\’ .  
LEFTARROW <− ’<−’ Spacing  
SLASH <− ’/’ Spacing  
AND <− ’&’ Spacing  
NOT <− ’!’ Spacing  
QUERY <− ’?’ Spacing  
STAR <− ’*’ Spacing  
PLUS <− ’+’ Spacing  
OPEN <− ’(’ Spacing  
CLOSE <− ’)’ Spacing  
DOT <− ’.’ Spacing  
Spacing <− ( Space / Comment )*  
Comment <− ’#’ ( !EndOfLine . )* EndOfLine  
Space <− ’ ’ / ’\t’ / EndOfLine  
EndOfLine <− ’\r\n’ / ’\n’ / ’\r’  
EndOfFile <− !.  
Action <− ’{’ < [^}]* > ’}’ Spacing  
BEGIN <− ’<’ Spacing  
END <− ’>’ Spacing

## LEG GRAMMARS<a name="LEG_GRAMMARS"></a>

_leg_ is a variant of _peg_ that adds some features of _lex_(1) and _yacc_(1). It differs from _peg_ in the following ways.**  
%{ **_text... _**%}**

A declaration section can appear anywhere that a rule definition is expected. The _text_ between the delimiters ’%{’ and ’%}’ is copied verbatim to the generated C parser code _before_ the code that implements the parser itself.

_name _**= **_pattern_

The ’assignment’ operator replaces the left arrow operator ’<−’.

**rule−name**

Hyphens can appear as letters in the names of rules. Each hyphen is converted into an underscore in the generated C source code. A single hyphen ’−’ is a legal rule name.

− = [ \t\n\r]*  
number = [0−9]+ −  
name = [a−zA−Z_][a−zA_Z_0−9]* −  
l−paren = ’(’ −  
r−paren = ’)’ −

This example shows how ignored whitespace can be obvious when reading the grammar and yet unobtrusive when placed liberally at the end of every rule associated with a lexical element.

_seq−1 _**| **_seq−2_

The alternation operator is vertical bar ’|’ rather than forward slash ’/’. The _peg_ rule

name <− sequence−1  
/ sequence−2  
/ sequence−3

is therefore written

name = sequence−1  
| sequence−2  
| sequence−3  
;

in _leg_ (with the final semicolon being optional, as described next).

_@{ action }_

Actions prefixed with an ’at’ symbol will be performed during parsing, at the time they are encountered while matching the input text with a rule. Because of back-tracking in the PEG parsing algorithm, actions prefixed with ’@’ might be performed multiple times for the same input text. (The usual behviour of actions is that they are saved up until matching is complete, and then those that are part of the final derivation are performed in left-to-right order.) The variable _yytext_ is available within these actions.

_exp _**~ **_{ action }_

A postfix operator **~**_{ action }_ can be placed after any expression and will behave like a normal action (arbitrary C code) except that it is invoked only when _exp_ fails. It binds less tightly than any other operator except alternation and sequencing, and is intended to make error handling and recovery code easier to write. Note that _yytext_ and _yyleng_ are not available inside these actions, but the pointer variable _yy_ is available to give the code access to any user−defined members of the parser state (see "CUSTOMISING THE PARSER" below). Note also that _exp_ is always a single expression; to invoke an error action for any failure within a sequence, parentheses must be used to group the sequence into a single expression.

rule = e1 e2 e3 ~{ error("e[12] ok; e3 has failed"); }  
| ...

rule = (e1 e2 e3) ~{ error("one of e[123] has failed"); }  
| ...

_pattern _**;**

A semicolon punctuator can optionally terminate a _pattern_.

**%% **_text..._

A double percent ’%%’ terminates the rules (and declarations) section of the grammar. All _text_ following ’%%’ is copied verbatim to the generated C parser code _after_ the parser implementation code.

**$$ = **_value_

A sub−rule can return a semantic _value_ from an action by assigning it to the pseudo−variable ’$$’. All semantic values must have the same type (which defaults to ’int’). This type can be changed by defining YYSTYPE in a declaration section.

_identifier_**:**_name_

The semantic value returned (by assigning to ’$$’) from the sub−rule _name_ is associated with the _identifier_ and can be referred to in subsequent actions.

The desk calculator example below illustrates the use of ’$$’ and ’:’.

## LEG EXAMPLE: A DESK CALCULATOR<a name="LEG_EXAMPLE_A_DESK_CALCULATOR"></a>

The extensions in _leg_ described above allow useful parsers and evaluators (including declarations, grammar rules, and supporting C functions such as ’main’) to be kept within a single source file. To illustrate this we show a simple desk calculator supporting the four common arithmetic operators and named variables. The intermediate results of arithmetic evaluation will be accumulated on an implicit stack by returning them as semantic values from sub−rules.

%{  
#include <stdio.h> /* printf() */  
#include <stdlib.h> /* atoi() */  
int vars[26];  
%}

Stmt = − e:Expr EOL { printf("%d\n", e); }  
| ( !EOL . )* EOL { printf("error\n"); }

Expr = i:ID ASSIGN s:Sum { $$ = vars[i] = s; }  
| s:Sum { $$ = s; }

Sum = l:Product  
( PLUS r:Product { l += r; }  
| MINUS r:Product { l −= r; }  
)* { $$ = l; }

Product = l:Value  
( TIMES r:Value { l *= r; }  
| DIVIDE r:Value { l /= r; }  
)* { $$ = l; }

Value = i:NUMBER { $$ = atoi(yytext); }  
| i:ID !ASSIGN { $$ = vars[i]; }  
| OPEN i:Expr CLOSE { $$ = i; }

NUMBER = < [0−9]+ > − { $$ = atoi(yytext); }  
ID = < [a−z] > − { $$ = yytext[0] − ’a’; }  
ASSIGN = ’=’ −  
PLUS = ’+’ −  
MINUS = ’−’ −  
TIMES = ’*’ −  
DIVIDE = ’/’ −  
OPEN = ’(’ −  
CLOSE = ’)’ −

− = [ \t]*  
EOL = ’\n’ | ’\r\n’ | ’\r’ | ’;’

%%

int main()  
{  
while (yyparse())  
;  
return 0;  
}

## LEG GRAMMAR FOR LEG GRAMMARS<a name="LEG_GRAMMAR_FOR_LEG_GRAMMARS"></a>

The grammar for _leg_ grammars is shown below. This will both illustrate and formalise the above description.

grammar = −  
( declaration | definition )+  
trailer? end−of−file

declaration = ’%{’ < ( !’%}’ . )* > RPERCENT

trailer = ’%%’ < .* >

definition = identifier EQUAL expression SEMICOLON?

expression = sequence ( BAR sequence )*

sequence = error+

error = prefix ( TILDE action )?

prefix = AND action  
| ( AND | NOT )? suffix

suffix = primary ( QUERY | STAR | PLUS )?

primary = identifier COLON identifier !EQUAL  
| identifier !EQUAL  
| OPEN expression CLOSE  
| literal  
| class  
| DOT  
| action  
| BEGIN  
| END

identifier = < [−a−zA−Z_][−a−zA−Z_0−9]* > −

literal = [’] < ( ![’] char )* > [’] −  
| ["] < ( !["] char )* > ["] −

class = ’[’ < ( !’]’ range )* > ’]’ −

range = char ’−’ char | char

char = ’\\’ [abefnrtv’"\[\]\\]  
| ’\\’ [0−3][0−7][0−7]  
| ’\\’ [0−7][0−7]?  
| !’\\’ .

action = ’{’ < braces* > ’}’ −

braces = ’{’ braces* ’}’  
| !’}’ .

EQUAL = ’=’ −  
COLON = ’:’ −  
SEMICOLON = ’;’ −  
BAR = ’|’ −  
AND = ’&’ −  
NOT = ’!’ −  
QUERY = ’?’ −  
STAR = ’*’ −  
PLUS = ’+’ −  
OPEN = ’(’ −  
CLOSE = ’)’ −  
DOT = ’.’ −  
BEGIN = ’<’ −  
END = ’>’ −  
TILDE = ’~’ −  
RPERCENT = ’%}’ −

− = ( space | comment )*  
space = ’ ’ | ’\t’ | end−of−line  
comment = ’#’ ( !end−of−line . )* end−of−line  
end−of−line = ’\r\n’ | ’\n’ | ’\r’  
end−of−file = !.

## CUSTOMISING THE PARSER<a name="CUSTOMISING_THE_PARSER"></a>

The following symbols can be redefined in declaration sections to modify the generated parser code.**  
YYSTYPE**

The semantic value type. The pseudo−variable ’$$’ and the identifiers ’bound’ to rule results with the colon operator ’:’ should all be considered as being declared to have this type. The default value is ’int’.

**YYPARSE**

The name of the main entry point to the parser. The default value is ’yyparse’.

**YYPARSEFROM**

The name of an alternative entry point to the parser. This function expects one argument: the function corresponding to the rule from which the search for a match should begin. The default is ’yyparsefrom’. Note that yyparse() is defined as

int yyparse() { return yyparsefrom(yy_foo); }

where ’foo’ is the name of the first rule in the grammar.

**YY_INPUT(**_buf_**, **_result_**, **_max_size_**)**

This macro is invoked by the parser to obtain more input text. _buf_ points to an area of memory that can hold at most _max_size_ characters. The macro should copy input text to _buf_ and then assign the integer variable _result_ to indicate the number of characters copied. If no more input is available, the macro should assign 0 to _result_. By default, the YY_INPUT macro is defined as follows.

#define YY_INPUT(buf, result, max_size) \  
{ \  
int yyc= getchar(); \  
result= (EOF == yyc) ? 0 : (*(buf)= yyc, 1); \  
}

Note that if YY_CTX_LOCAL is defined (see below) then an additional first argument, containing the parser context, is passed to YY_INPUT.

**YY_DEBUG**

If this symbols is defined then additional code will be included in the parser that prints vast quantities of arcane information to the standard error while the parser is running.

**YY_BEGIN**

This macro is invoked to mark the start of input text that will be made available in actions as ’yytext’. This corresponds to occurrences of ’<’ in the grammar. These are converted into predicates that are expected to succeed. The default definition

#define YY_BEGIN (yybegin= yypos, 1)

therefore saves the current input position and returns 1 (’true’) as the result of the predicate.

<table rules="none" frame="void" border="0" cellpadding="0" cellspacing="0" width="100%">

<tbody>

<tr align="left" valign="top">

<td width="9%">

**YY_END**

</td>

<td width="78%">

This macros corresponds to ’>’ in the grammar. Again, it is a predicate so the default definition saves the input position before ’succeeding’.

</td>

</tr>

</tbody>

</table>

#define YY_END (yyend= yypos, 1)

**YY_PARSE(**_T_**)**

This macro declares the parser entry points (yyparse and yyparsefrom) to be of type _T_. The default definition

#define YY_PARSE(T) T

leaves yyparse() and yyparsefrom() with global visibility. If they should not be externally visible in other source files, this macro can be redefined to declare them ’static’.

#define YY_PARSE(T) static T

**YY_CTX_LOCAL**

If this symbol is defined during compilation of a generated parser then global parser state will be kept in a structure of type ’yycontext’ which can be declared as a local variable. This allows multiple instances of parsers to coexist and to be thread−safe. The parsing function _yyparse_() will be declared to expect a first argument of type ’yycontext *’, an instance of the structure holding the global state for the parser. This instance must be allocated and initialised to zero by the client. A trivial but complete example is as follows.

#include <stdio.h>

#define YY_CTX_LOCAL

#include "the−generated−parser.peg.c"

int main()  
{  
yycontext ctx;  
memset(&ctx, 0, sizeof(yycontext));  
while (yyparse(&ctx));  
return 0;  
}

Note that if this symbol is undefined then the compiled parser will statically allocate its global state and will be neither reentrant nor thread−safe. Note also that the parser yycontext structure is initialised automatically the first time _yyparse_() is called; this structure **must** therefore be properly initialised to zero before the first call to _yyparse_().

**YY_CTX_MEMBERS**

If YY_CTX_LOCAL is defined (see above) then the macro YY_CTX_MEMBERS can be defined to expand to any additional member field declarations that the client would like included in the declaration of the ’yycontext’ structure type. These additional members are otherwise ignored by the generated parser. The instance of ’yycontext’ associated with the currently−active parser is available within actions as the pointer variable _yy_.

**YY_BUFFER_SIZE**

The initial size of the text buffer, in bytes. The default is 1024 and the buffer size is doubled whenever required to meet demand during parsing. An application that typically parses much longer strings could increase this to avoid unnecessary buffer reallocation.

**YY_STACK_SIZE**

The initial size of the variable and action stacks. The default is 128, which is doubled whenever required to meet demand during parsing. Applications that have deep call stacks with many local variables, or that perform many actions after a single successful match, could increase this to avoid unnecessary buffer reallocation.

**YY_MALLOC(**_YY_**, **_SIZE_**)**

The memory allocator for all parser−related storage. The parameters are the current yycontext structure and the number of bytes to allocate. The default definition is: malloc(_SIZE_)

**YY_REALLOC(**_YY_**, **_PTR_**, **_SIZE_**)**

The memory reallocator for dynamically−grown storage (such as text buffers and variable stacks). The parameters are the current yycontext structure, the previously−allocated storage, and the number of bytes to which that storage should be grown. The default definition is: realloc(_PTR_, _SIZE_)

**YY_FREE(**_YY_**, **_PTR_**)**

The memory deallocator. The parameters are the current yycontext structure and the storage to deallocate. The default definition is: free(_PTR_)

**YYRELEASE**

The name of the function that releases all resources held by a yycontext structure. The default value is ’yyrelease’.

The following variables can be referred to within actions.**  
char *yybuf**

This variable points to the parser’s input buffer used to store input text that has not yet been matched.

**int yypos**

This is the offset (in yybuf) of the next character to be matched and consumed.

**char *yytext**

The most recent matched text delimited by ’<’ and ’>’ is stored in this variable.

**int yyleng**

This variable indicates the number of characters in ’yytext’.

**yycontext *yy**

This variable points to the instance of ’yycontext’ associated with the currently−active parser.

Programs that wish to release all the resources associated with a parser can use the following function.**  
yyrelease(yycontext**_*_**yy**_)_

Returns all parser−allocated storage associated with _yy_ to the system. The storage will be reallocated on the next call to _yyparse_().

Note that the storage for the yycontext structure itself is never allocated or reclaimed implicitly. The application must allocate these structures in automatic storage, or use _calloc_() and _free_() to manage them explicitly. The example in the following section demonstrates one approach to resource management.

## LEG EXAMPLE: EXTENDING THE PARSER’S CONTEXT<a name="LEG_EXAMPLE_EXTENDING_THE_PARSERS_CONTEXT"></a>

The _yy_ variable passed to actions contains the state of the parser plus any additional fields defined by YY_CTX_MEMBERS. Theses fields can be used to store application−specific information that is global to a particular call of _yyparse_(). A trivial but complete _leg_ example follows in which the yycontext structure is extended with a _count_ of the number of newline characters seen in the input so far (the grammar otherwise consumes and ignores the entire input). The caller of _yyparse_() uses _count_ to print the number of lines of input that were read.

%{  
#define YY_CTX_LOCAL 1  
#define YY_CTX_MEMBERS \  
int count;  
%}

Char = (’\n’ | ’\r\n’ | ’\r’) { yy−>count++ }  
| .

%%

#include <stdio.h>  
#include <string.h>

int main()  
{  
/* create a local parser context in automatic storage */  
yycontext yy;  
/* the context *must* be initialised to zero before first use*/  
memset(&yy, 0, sizeof(yy));

while (yyparse(&yy))  
;  
printf("%d newlines\n", yy.count);

/* release all resources associated with the context */  
yyrelease(&yy);

return 0;  
}

## DIAGNOSTICS<a name="DIAGNOSTICS"></a>

_peg_ and _leg_ warn about the following conditions while converting a grammar into a parser.**  
syntax error**

The input grammar was malformed in some way. The error message will include the text about to be matched (often backed up a huge amount from the actual location of the error) and the line number of the most recently considered character (which is often the real location of the problem).

**rule ’foo’ used but not defined**

The grammar referred to a rule named ’foo’ but no definition for it was given. Attempting to use the generated parser will likely result in errors from the linker due to undefined symbols associated with the missing rule.

**rule ’foo’ defined but not used**

The grammar defined a rule named ’foo’ and then ignored it. The code associated with the rule is included in the generated parser which will in all other respects be healthy.

**possible infinite left recursion in rule ’foo’**

There exists at least one path through the grammar that leads from the rule ’foo’ back to (a recursive invocation of) the same rule without consuming any input.

Left recursion, especially that found in standards documents, is often ’direct’ and implies trivial repetition.

# (6.7.6)  
direct−abstract−declarator =  
LPAREN abstract−declarator RPAREN  
| direct−abstract−declarator? LBRACKET assign−expr? RBRACKET  
| direct−abstract−declarator? LBRACKET STAR RBRACKET  
| direct−abstract−declarator? LPAREN param−type−list? RPAREN

The recursion can easily be eliminated by converting the parts of the pattern following the recursion into a repeatable suffix.

# (6.7.6)  
direct−abstract−declarator =  
direct−abstract−declarator−head?  
direct−abstract−declarator−tail*

direct−abstract−declarator−head =  
LPAREN abstract−declarator RPAREN

direct−abstract−declarator−tail =  
LBRACKET assign−expr? RBRACKET  
| LBRACKET STAR RBRACKET  
| LPAREN param−type−list? RPAREN

## CAVEATS<a name="CAVEATS"></a>

A parser that accepts empty input will _always_ succeed. Consider the following example, not atypical of a first attempt to write a PEG−based parser:

Program = Expression*  
Expression = "whatever"  
%%  
int main() {  
while (yyparse())  
puts("success!");  
return 0;  
}

This program loops forever, no matter what (if any) input is provided on stdin. Many fixes are possible, the easiest being to insist that the parser always consumes some non−empty input. Changing the first line to

Program = Expression+

accomplishes this. If the parser is expected to consume the entire input, then explicitly requiring the end−of−file is also highly recommended:

Program = Expression+ !.

This works because the parser will only fail to match ("!" predicate) any character at all ("." expression) when it attempts to read beyond the end of the input.

## BUGS<a name="BUGS"></a>

You have to type ’man peg’ to read the manual page for _leg_(1).

The ’yy’ and ’YY’ prefixes cannot be changed.

Left recursion is detected in the input grammar but is not handled correctly in the generated parser.

Diagnostics for errors in the input grammar are obscure and not particularly helpful.

The operators **! **and **~** should really be named the other way around.

Several commonly−used _lex_(1) features (yywrap(), yyin, etc.) are completely absent.

The generated parser does not contain ’#line’ directives to direct C compiler errors back to the grammar description when appropriate.

## SEE ALSO<a name="SEE_ALSO"></a>

D. Val Schorre, _META II, a syntax−oriented compiler writing language,_ 19th ACM National Conference, 1964, pp. 41.301−−41.311\. Describes a self−implementing parser generator for analytic grammars with no backtracking.

Alexander Birman, _The TMG Recognition Schema,_ Ph.D. dissertation, Princeton, 1970\. A mathematical treatment of the power and complexity of recursive−descent parsing with backtracking.

Bryan Ford, _Parsing Expression Grammars: A Recognition−Based Syntactic Foundation,_ ACM SIGPLAN Symposium on Principles of Programming Languages, 2004\. Defines PEGs and analyses them in relation to context−free and regular grammars. Introduces the syntax adopted in _peg_.

The standard Unix utilities _lex_(1) and _yacc_(1) which influenced the syntax and features of _leg_.

The source code for _peg_ and _leg_ whose grammar parsers are written using themselves.

The latest version of this software and documentation:

http://piumarta.com/software/peg

## AUTHOR<a name="AUTHOR"></a>

_peg_, _leg_ and this manual page were written by Ian Piumarta (first−name at last−name dot com) while investigating the viability of regular and parsing−expression grammars for efficiently extracting type and signature information from C header files.

Please send bug reports and suggestions for improvements to the author at the above address.

* * *
