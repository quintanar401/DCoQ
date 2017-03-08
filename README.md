# Dark corners of Q

### Intro

There are many hidden features and obscure practices in Q. This is the list of some of them.

### Magic value

The magic value is something that looks like the identity function `(::)` but creates void everywhere. The magic value can be created from any projected function with a missing argument:
```
q)mv:(value(1;))2;
```
At first sight it looks like the identity
```
q)string mv
"::"
q)mv[1]
1
q)1 2[mv]
1 2
```
But only until you try it in more complex expressions and then the weird things start to happen. You may think you can't add `(::)` and a number but wait:
```
q)mv + 10
+[;10]
```
And you can add mv to mv too
```
q)mv + mv
+[;]
```
mv represents some kind of void - if it appears in the function's argument list this argument will be omitted and the function will become a projection
```
q){z}[mv;"1";"2"]
{z}[;"1";"2"]
```
There are some limitations though - an unary function can't have a projection so it gets evaluated as usual
```
q)null mv
0b
```
mv is difficult to handle because it tends to project all functions around. For example it is not easy to create a list of mvs
```
q)10#mv
#[10;]
```
Fortunately we can use unary functions that are immune to projection
```
q)l:raze 10#enlist mv
q)l[1]+l[2]
+[;]
```
And now we are ready to do something that is extremely difficult to do in the boring ordinary Q without magic - project functions via apply!
```
q){z} . (;1;2) / will not work!
q){z} . @[3#l;1 2;:;1 2] / works!
{z}[;1;2]
```
We can even project functions dynamically
```
q)args:0 2!(10;20)
q)prj:{x . @[(count value[x]1)#l;key y;:;value y]}
q)prj[{z};args]
{z}[10;;20]
```
An important special case - enlist function. We can't use apply because it checks that the number of arguments is less than 9 but we can use eval
```
q)l:100#l
q)prj2:{eval(enlist),@[x#l;key y;:;value y]}
q)prj2[50;args]
enlist(10;;20;;;....;;)
```
It's not easy to create such enlist with arbitrary values at random places! Lets use it to calculate some aggregation function on a list with a random number of nulls that will be filled later
```
q)v:100?til[10],1#l / add some mv nulls
q)f:(')[max;eval(enlist),v] / create func, f now is a function of a random number of args!
q){x first 1?100}/[{100<type x};f] / feed random numbers to f until it is done
65
```

### ";" as an identity function

It may look crazy but ";" indeed acts like an identity function
```
q)";"[1]
1
q)";"[{x+y}]
{x+y}
```
Even more - it is actually the last identity function because you can pass to it any number of arguments (more than 8!)
```
q)";"[1;2;3]
3
```
It is immune to projections and to the magic value and can't be used with apply
```
q)";"[;mv;10]
10
```
Strange properties of ";" become clear if we notice that
```
q)parse "1;2;3"
(";";1;2;3)
```
it is just the block marker in the eval tree. Actually all 3 expressions are equivalent
```
1;2;3
[1;2;3]
";"[1;2;3]
```

### "q" and "k" are functions

Like ";" "q" and "k" are also special functions. You can use them instead of value to evaluate string expressions
```
q)"q" "1+1"
2
q)"k" ",1"
,1
```
"k" is especially useful if you want to evaluate k code and can't use "k)" syntax
```
q)("k" ",#:")1 2 3
,3
```
Actually any char can be a function (except ";"), all you have to do
```
q)(`$".!.e")set {[s] value s}
q)"!" "1+1"
2
```
is to provide an eval function. More on this feature in DSL section.

### Mistery of :

You may think that ";" and "k" functions are weird but there's more
```
q)a:{x+1}
q)(a:) 100
101
```
Yes `a:` is the same as just `a`, actually you can add as many ":" as you want, it seems Q doesn't care how many ":" follow a symbol
```
q)(a:::::::::)100
101
```
What happens here is that this expression really means this
```
(a:::::)~(::)a~(:)a
```
Looks like it is K leaking into Q because you can call other functions in this manner, also you can use other expression delimiters
```
q)(10!:)
0 1 2 3 4 5 6 7 8 9
q)[10!:]
q)b:10!:;
```
Unfortunately these unary functions can't be chained, they require some Q expression on the left and execute immediately. There are some tricks though.
First of all you can use them to avoid some verbose Q functions like string or enlist
```
q)(`mysym$:; 10,:)
("mysym";,10)
/ looks especially good when 1 item lists are mixed with multi item lists, adds uniformity to the expression
q)("ab";"a",:)
```
Also they provide an excelent way to define a composite function - just put :: at the end
```
q)a:{x+y}; b:{x+1}
q)c:{x*2}b a ::
q)c[10;20]
62
```
Notice that `::` syntax is valid for functions with any number of arguments (unlike `f g@`) and is simpler than `'[f;g]`.
If your first function in a composition is one of ":" functions you can use the following trick
```
q)c:{x+1}{x*2}:: !:
q)c 3
1 3 5
```
You can use two ":" functions at the end. In this case one of them gets applied to its left argument and the second (til in this example) becomes the first function of the composition.

### Arbitrary staff out of comments

All Q expressions start from the new line with no spaces before them. This is a common mistake to omit spaces inside a multiline function or before the last curly bracket
```
f:{ / expresion starts
 1+1; / it is ok
2+2 / wrong, 2+2 starts the new expression thus f is incorrect
} / also wrong, every line including this must be indented
```
What if there is no start line like when it is the start of a file. Then all indented lines at the top are ignored
```
  This is the first line so I don't need to comment it
  and the line after it
  any number of lines until
a:100; / the first real expression that has no whitespace before it
```
Some people use this feature to provide help about functions in the file. Comments and empty lines get deleted before expressions are evaluated so you can mix them with indented lines at the top
```
/ Ordinary comment
 can be continued
 on and on
a:100 / until the first expression
```
There is another example when you can comment without a comment
```
\d .
 d syscmd doesn't mind if there is
 something after it but
 it switches to some namespace based on this comment so
a:100 / anything defined after it is lost in limbo
.c.c:0N!a+100 / but while you are in this context you can use defined vars, side effects and globals
system "d" / reports . (or whatever is used with d) but it will be inaccessible from outside
.c.f:{a::a+1} / but the namespace is still here, you can execute .c.f[]
\d .
b:100 / you must repeat d to resume the correct processing
```
I guess this feature can be used for private namespaces that can't be changed but via public interface functions saved in another namespace.

### A way to get local variables by name in a function

We know that it is impossible to use `get` ( `value` ) or any other function to obtain the value of a local variable by name
```
{b:1; get `b}
'b
```
But there is one (or two, but we need only one) exception, only one function that has access to the local variables of the function that called it - it is `select` (`?` more exactly). All we need to do is to provide a dummy dictionary
```
q)dummy:(`$())!()
q)f:{b1:10;b2:20; ?[dummy;();();x]}
q)f `b1
10
q)f `b2
20
```
Unfortunately it is impossible to make a function from this construction - any projection will remove local variables from the exec scope. If you need to `get` a local variable you'll have to write this exec every time.

You can use this feature to debug any function by intercepting its arguments
```
q)f:{[a]v:100; `DEBUG; a+v}; / first we add `DEBUG expression that has no effect on the function
/ preprocess function to add the monitoring expression - 0N! here but it can be any function
q)m:{value ssr[string x;"`DEBUG";"0N!?[dummy;();();{x!x}raze(value .z.s)1 2]"]}
q)f:m f; / preprocess f, you may preprocess the whole file
q)f 1
`a`v!1 100
101
```
You don't need to know the function's arguments beforehand thus you are able to preprocess files as text before they are loaded or intercept arguments in functions defined in functions.

### Subvert timeouts via 0 handle and catch blocks

Sometimes it is useful to avoid timeout set on a server. For example if another server calls a system function. Also users can accidentally supress timeout and it is useful to known how this
can happen.

The first way is to set it to 0 and execute a function via 0 handle
```
q)\T 1
q)0 "do[10000000;exp 1.1]"
'stop
q){t:system "T"; system "T 0"; 0 "do[10000000;exp 1.1]"; system "T ",string t}[] / runs ok
```
The second way is to catch `'stop` and continue as if nothing happened (users can accidentally do this)
```
q)0 ({n:0;while[n<3;0N!n+:1;@[{do[10000000;exp 1.1]};::;{0N!x}]]};::)
1
"stop"
2
3
```
As you can see `'stop` is triggered only once so the code after the catch block can run forever.

### Functions with unlimited number of args

With the magic value it is easy to have functions with unlimited number of arguments. The core function will have to accept a list of arguments of course but from the outside the function will look like an ordinary function but with more than 8 args.
```
q)ml:(value (;))(),1
q)mkfn:{'[x;eval(enlist),y#ml]}
q)megasum: mkfn[sum;10]
q)f:megasum[1;2;3;4] / set first 4 values
q)f . til 6 / apply other 6
25
```
As you can see we can create projections, use apply and the mega function will behave as an ordinary function.

We can also create functions with variable number of arguments, in this case we just use enlist as the first function
```
q)megasum:(')[sum;enlist]
q)megasum[1;2;3;4;5;6;7;8;9]
45
```
Because the number of args is variable the function will be executed as soon as there is at least 1 argument and there are no unfilled projections
```
q)megasum[1;;2][3]
6
```

### DSLs on the fly

Q has built-in support for DSL languages - q itself is a good example. Another example that comes from KX is SQL92 subset language that can be found in s.k file.

You can create your own language too. Actually it is very easy, all you have to do is to define .X.e (where X is some char) function that accepts a string and does something useful with it.

For example we can calculate the number of symbols in a string
```
q).c.e:{count x}
q)c)Let's see if it works
21
```
You can evaluate expressions in your language in many ways, one of them is to add X) at the start of a line. `value` will accept it too
```
q)value "c)value is a great function"
25
```
Less usual but also acceptable - use "X" as a function
```
q)"c""it's ok to do this"
18
```
Finally you can create any_file.X in your DSL and then load it via `l`
```
q)`:test.c 0: ("Let's see";"if it works")
q)\l test.c
9
11
```
Note that Q sticks to its idiosyncrasy with indentation and enforces its own comments
```
q)`:test.c 0: ("Let's see";" if it works";"/ comments are still Q")
q)\l test.c
22
```
Good way to get the preprocessed Q code - copy any.q to any.X and in .X.e you'll get `value` ready Q code.

Worth to note - if .X.e function is not defined Q will try to load "X.q" or "X.k" when you try one of the above expressions.

### Lexer in 5 minutes

With scan function and a matrix we can create a simple lexer pretty quick. An example can be found in s.k KX library. It is written in K and is quite cryptic so I provide here a small library to create lexer matrices in a more straightforward way.

I'll create a lexer for a subset of Q. First we need to divide all chars into groups for convenience and list them from more general to more specific
```
sg:("\t \r\n";"0123456789";.Q.a,.Q.A;"<>";"+-";",*%&|$?#!~^"); / multi symbol groups
sg:sg,"e._:=/\\\"`'"; / 1 symbol groups
c2g:@[128#0;"j"$sg;:;1+til count sg]; / char to group map
```
Then we define a state transition map `states->new_state onGroupsOfChars`. The small letter and special char states are initial states corresponding to the char groups, the capital letter states
are token states
```
/ A - id, B - dead end, C -  sym + :,  EF - numbers,floats,times,dates, KL -strings
mst:" "vs/:("aeA A ae0._"; ". A ae"; / identifiers
  ". E 0";"0E E 0.:a";"0E F e";"F E +0ae."; / Q lexer doesn't care about the exact form of a number/float/date/stamp, it parses first then checks
  "< B ="; "/\\'0:+.=<_,C C :"; / <= and >= and ::::: /: \: ': ....
  "` S ae0.";"S S ae0.:_";"` T :";"T T ae0.:_/"; / symbols and paths
  "\" K *"; "K K *"; "K L \""; "K M \\"; "\" L \""; "\" M \\"; "M K *"; / strings, K-start/middle, L-end, M-\X
  "\tW W \t" / merge ws for convenience
 );
```
We create a state matrix from these rules
```
st:distinct" ",(first each sg),raze mst[;1]; / all matrix states, before A - initial states
mt:{a:st?y;x[a 0;(a 2;::)"*"in y 2]:first a 1;x}/[count[st]#enlist til count sg;mst]
```
The lex function itself is simple
```
lx:{(where(mt\[0;c2g x])<st?"A")cut x}

q)lx string lx
(,"{";,"(";"where";,"(";,"c";,":";"mt";,"\\";,"[";,"0";,";";"c2g";," ";,"x";,"]";,")";,"<";"st";,"?";"\"A\"";,")";"cut";," ";,"x";,"}")
```
The token type is lost in lx but we can calculate it if we also cut the matrix result, apply max to each token and define a map from states to token names
```
s2t:{(!). flip raze(st?x[;0]),\:'x[;1]}(("aeA";`ident);("0EF";`num);("`ST";`sym);("KLM";`str);("\tW";`ws);("C",:;`specfn));
lx2:{flip`tok`txt!(s2t max each i cut c;(i:where(c:mt\[0;c2g x])<st?"A")cut x)}
q)lx2 string lx2
tok   txt       
----------------
      enlist "{"
ident "flip"    
sym   "`tok"    
sym   "`txt"
...
```
