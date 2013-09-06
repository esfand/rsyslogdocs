### Expressions ###

The language supports arbitrary complex expressions. All usual operators are supported. The precedence of operations is as follows (with operations being higher in the list being carried out before those lower in the list, e.g. multiplications are done before additions.

    expressions in parenthesis
    not, unary minus
    *, /, % (modulus, as in C)
    +, -, & (string concatenation)
    ==, !=, <>, <, >, <=, >=, contains (strings!), startswith (strings!)
    and
    or

For example, "not a == b" probably returns not what you intended. The script processor will first evaluate "not a" and then compare the resulting boolean to the value of b. What you probably intended to do is "not (a == b)". And if you just want to test for inequality, we highly suggest to use "!=" or "<>". Both are exactly the same and are provided so that you can pick whichever you like best. So inquality of a and b should be tested as "a <> b". The "not" operator should be reserved to cases where it actually is needed to form a complex boolean expression. In those cases, parenthesis are highly recommended.


Rsyslog supports expressions at a growing number of places. 
So far, they are supported for filtering messages.

C-like comments `/* some comment */` are supported inside the expression, 
but not yet in the rest of the configuration file.
