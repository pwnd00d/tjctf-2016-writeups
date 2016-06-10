#guess

In this problem, you are to guess a random number 100 times in a row. When I did this problem, I didn't realize it gave the source or even the binary, but a simple guess made it fairly quick to do.

My first train of thought was to do some sort of buffer overflow, but I quickly realized I wasn't getting anywhere with that, so I guessed the random number generator was seeded with something, like maybe the time.

So I tested this by making a first writing a short program to get the 'current' random number based on the current time in C:

```c
#include<stdio.h>
#include<time.h>
#include<stdlib.h>

int main(void)
{
 int rand_num;
 srand(time(0)); //seed with current time

  rand_num = rand();
  printf("%d", rand_num);
 
 return 0;
}
```

Then I made a python script that ran that program via subprocess and immediately sent the output to the server, repeating 100 times:

```python
import socket,subprocess

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('p.tjctf.org', 8007))
data = s.recv(1024)
print data

for i in range(100):
    args = ("./guess.c.o")
    popen = subprocess.Popen(args, stdout=subprocess.PIPE)
    popen.wait()
    output = popen.stdout.read()
    print(output)
    s.send(output+"\n")
    data = s.recv(1024)
    print data
data = s.recv(1024)
print data
```

So this works, but it seems that it stops somewhere in the middle most of the time, and I assumed that this was because of latency, that the request was made in a different second than the response. The chance that this occurs is small enough that I can get it 100 times in a row after a few tries, which results in the flag: 
```
tjctf{n0t_so_r4ndom_4nymor3}
```
