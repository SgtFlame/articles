# ZScript for Meta Zen

Originally ZScript was designed for use with Zen Spaces, but as I progressed with ZScript and Meta Zen, I realized they were a great combination.

## Enhancements

While ZScript has a lot of functionality, one thing that is missing is the manipulation of scripts without requiring a full recompile.

The way I see this happening is a script is loaded in memory, and then through a GUI or command line (or both), the script is manipulated without breaking the reactive pub/sub chain.  The modified code marks the chain dirty just as it would if source data were modified.

I see this being done through something like JQuery (and possibly even through the use of JavaScript as the primary manipulation language).

Very simple example:
```
zs.def('x', 10);
=> (def x 10)
zs.find('x').def('12');
=> (def x 12)
```

Replace a function with a wrapped function:
```
zs.find('x').wrap('add 1 ?');
=> (def x (add 1 12)
zs.find('x').wrap('sub ? 2');
=> (def x (sub (add 1 12) 2))
```

Replace an argument, either by finding the argument by the  name of the function being evaluated to define the argument, or by the argument index ($0 is function call, $1 is first argument, etc)
```
var add = zs.find('x add').replace('+ 1 12');
# Equivalent to 
var add = zs.find('x').$1.replace('+ 1 12');
=> (def x (sub (+ 1 12) 2 ))
=> add = (add 1 12)

add.$2.replace(2);
=> add = (add 1 2)

zs.find('x sub').$2.replace(add);
=> (def x (sub (+ 1 12) (add 1 12)))
```
