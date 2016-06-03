# イヤホンと蝉時雨 (Earphones and a Chorus of Cicadas)

In this problem, we're given [cicadas.txt](cicadas.txt), a file that contains some instructions at the top and then a series of strange equations. We're being told that every letter represents a digit, and that uppercase and lowercase letters may represent different numbers.

As soon as I saw this problem, I realized this would be a task for [z3](http://rise4fun.com/z3), a theorem prover. To put it simply, this can solve systems of equations very quickly. Essentially, what we'd need to do is turn all the equations from the text file we were given originally into a set of executable commands for z3, to crank out all the numbers.

I like Python, and luckily for me, z3 has a Python API. If you're following along, this is the part where you'll probably want to have z3 installed to continue. I wrote a script to generate a script that uses z3 to find the unique solution that satisfies all the equations. My first script, `generate.py`, reads the lines from the problem file and turns it into Python code and spits it into `solve.py`, which I then run to get my final solution.

So what does z3 look like? Well, here's a pretty basic example:

```python
from z3 import *
x = Int("x")

solver = Solver()
solver.add(x + 2 == 5)

if solver.check() == sat:
    print solver.model()
```

Not too complicated, right? In the line `x = Int("x")` I'm creating a variable `x`, that it's going to solve for. Then I created a `Solver`, and added a single equation, `x + 2 == 5` to it. Finally, I `check`ed if the system was `sat`isfiable, meaning that there was indeed a solution, and if so, print the `model`. If you run the code above, you should get some output that looks like this:

```
[x = 3]
```

Ok, so now that we know z3 works on a simpler example, let's go about creating a solve script for our problem. For this task, I've deleted the first few lines of `cicadas.txt` so all the remains are the equations. Let's start with:

```python
import string

data = open("cicadas.txt").read().split("\n")
out = open("solve.py", "w")

letters = string.uppercase + string.lowercase
```

Recall that the variables in the system consist of uppercase and lowercase letters. Using these letters, we will construct z3 variables.

```python
out.write("""
from z3 import *

solver = Solver()
%s = BitVecs('%s', 32)
""" % (", ".join(letters), " ".join(letters)))
```

I'm writing this to `solve.py`, so this must be executable Python code. Did you notice that I'm using `BitVecs` rather than `Int`s? The reason for this is that our equations involve operations such as `^` (bitwise XOR), `&` (bitwise AND), and `|` (bitwise OR) that `Int`s just can't do. Using `BitVecs`, however, we can perform all of these bitwise operations as well as standard operations.

But we can't just print back all the equations, because in our original equation, we might have a block of letters like `IONBGmcquy`. The z3 solver doesn't know to automatically treat that as a number where letters are substituted with digits. We must convert that into a form like

```python
1 * y + 10 * u + 100 * q + 1000 * c + 10000 * m + 100000 * G + 1000000 * B + 10000000 * N + 100000000 * O + 1000000000 * I
```

Notice that this expression is equivalent to the above block, except that it's valid Python syntax. We can write a function to convert every block of letters into expressions like the one above. Here's what this function looks like:

```python
def generate_expression(s):
    expression = []
    for i in range(len(s)):
        expression.append("%d * %s" % (pow(10, i), s[::-1][i]))
    return " + ".join(expression)
```

Here's the fun part: actually parsing all the equations. Each equation is conveniently divided by spaces. Let's split them using spaces as the delimiter:

```python
for line in data:
    parts = line.split(" ")
```

Each part could be one of three things:

- A block of letters. We'll use `generate_expression` to convert this into an expression.
- An operation. This includes `+`, `^`, `|`, and `&`. We can directly stick this into the equation, although recall that in the instructions, we're given that these equations should be evaluated from left to right, rather than in order of operations. This makes parsing much easier on our part, as we can just wrap the whole expression with parentheses `()` and append our operation.
- The equals sign. Since the equals sign always precedes the last block of letters, we can simply group the last block of letters together with this equals sign.

So what does this look like? This:

```python
for line in data:
    parts = line.split(" ")
    equation = ""
    for part in parts:
        if all([c in letters for c in part]):
            equation += generate_expression(part)
        elif part in "^|&+":
            equation = "(" + equation + ") " + part
        elif part == "=":
            equation += " == " + generate_expression(parts.pop(-1))
    out.write("solver.add(%s)\n" % equation)
```

The last line prints the equation we generated into the solve script. But the equations aren't all. Every letter represents a digit. That means no letter can take on a value greater than 10. We can add a couple more equations to fix this.

```python
for letter in letters:
    out.write("solver.add(%s < 10)\n" % letter)
```

Also, the trivial solution (all equations = 0) will fit the system. But that's not what we're looking for. So we need to ensure that at least one of the variables is not 0. The best way to do this is to force z3 to add all the variables together and make sure the sum is greater than 0.

```python
out.write("solver.add(%s > 0)\n" % (" + ".join(letters)))
```

Finally, we'll tell the solver to check if the system is satisfiable, and print the solution:

```python
out.write("""
if solver.check() == sat:
    print solver.model()
""")
```

Now we'll just let that run and sit there for an hour or so. The solution is:

```
[w = 4, e = 8, o = 5, P = 5, D = 7, f = 5, q = 1, Y = 0, E = 6, M = 4, z = 9, S = 2, H = 4, K = 8, t = 9, G = 1, J = 8, u = 7, j = 0, X = 3, b = 0, m = 8, B = 6, O = 7, F = 3, C = 9, N = 9, d = 1, n = 0, V = 0, U = 4, i = 7, s = 3, g = 1, A = 2, I = 8, R = 2, T = 6, p = 6, y = 7, k = 4, h = 2, Q = 0, a = 9, l = 3, L = 5, Z = 1, v = 6, c = 5, W = 1, x = 3, r = 2]
```

From here, the rest is trivial. Simply substitute the letters into the flag, and parse the flag as decimal ASCII values. I wrote a script to do this as well.

```python
import string
letters = string.uppercase + string.lowercase

data = "w = 4, e = 8, o = 5, P = 5, D = 7, f = 5, q = 1, Y = 0, E = 6, M = 4, z = 9, S = 2, H = 4, K = 8, t = 9, G = 1, J = 8, u = 7, j = 0, X = 3, b = 0, m = 8, B = 6, O = 7, F = 3, C = 9, N = 9, d = 1, n = 0, V = 0, U = 4, i = 7, s = 3, g = 1, A = 2, I = 8, R = 2, T = 6, p = 6, y = 7, k = 4, h = 2, Q = 0, a = 9, l = 3, L = 5, Z = 1, v = 6, c = 5, W = 1, x = 3, r = 2".split(", ")
key = {}
for item in data:
    parts = item.split(" = ")
    key[parts[0]] = parts[1]

flag = "gWT ZYv za gdT WbR ZSX ZZa Ghr WWZ gbQ ZZJ te GRd Nz dnF gjI gdk gYH GYz dAA GrQ ZZb ZnC GWT Ggg Zbv dnc Wqq qrh WqO gbU grQ aa gVP dGT GnL gZE gQl dbE qZH ay Wqj dqM tu gZB gdo gds zi WYq qji Gbm qjt GWx qZF Zrn dnT gQN qQa gVd WGu qGX dgq ZQQ ZVf ZZs ZVt ty aK aO gQz dbO gYo ZWJ qgi dQD gjZ qVs dGY drd qWh dhd qrQ GGF zD Wnv qbW qhA gRZ dby ZqW gQB dGg Gqj WZl ZYY dGK WQo WAW ZQU qYL Zde ZWn zm dVL dVE Zgo ZSP"
new_flag = ""
for c in flag:
    if c in letters:
        new_flag += key[c]
    else:
        new_flag += c

print "".join([chr(int(c)) for c in new_flag.split(" ")])
```

That's all! The flag is:

```
tjctf{wzodvbycglrhmzxnmtojiozuhxcititgjranratsqaeklmqqxjmmeuqodiqmabamkivukegnypyxqajezykojonqdviyhivnbijs}
```