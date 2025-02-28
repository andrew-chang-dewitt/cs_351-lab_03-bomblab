bomblab
=======

Andrew Chang-DeWitt \
Systems Programming \
CS 351 - Fall 2025 \
IIT

setup
-----

go to prj dir in terminal.
get some helpful info to explore in vim while running program in debugger: `objdump -d bomb.c > bomb.asm`.
note fn names for phases & explode_bomb seems important

then get started debugging with the following:
setup breakpoints for all 6 phases + secret phase & another on explode_bomb as safeguard to prevent explosions.
finally, run program w/ no args until first breakpoint on first phase.

```
gdb ./bomb
break <FN_NAME> -- repeat for all functions noted when exploring objdump output
run
```

should stop at phase 1


phase 1
-------

examine bomb.asm @ `phase_1`
note call to compare strings for equality, w/ first arg being program input & 2nd arg being supplied by program

set breakpoint to stop in comparison function using `b strings_not_equal`, then get actual & expected strings via:

```
print (char*)$rdi (user input)
print (char*)$rsi (program string) -- gives expected string
```

save output of program string as first line in `./inputs.txt`
remove `phase_1` breakpoint w/ `disable breakpoints <N>` (get breakpoint number w/ `info break` as it should be solved now

phase 2
-------

> [!TODO]:
> - [x] read comments in phase 2 fn from bomb.asm
> - [ ] write this section...

examining the `objdump` output for `phase_2` shows a call to `read_six_numbers` after first making room for 7 4-byte ints on the stack.
after stepping over the read six function, it can be observed that all six numbers from the input string were read into the stack, starting w/ `%rsp`.

after that, we encounter the first check that might explode the bomb, making sure that numbers were even loaded from input in the first place.
to do this, `phase_2` checks that rsp now contains a positive number (the value at the memory location pointed to by `%rsp` was negative before calling `read_six_numbers`).

```objdump
  # check that there are numbers loaded into the stack
  401044:	83 3c 24 00          	cmpl   $0x0,(%rsp)
  # if result was negative, then no numbers were loaded
  401048:	79 05                	jns    40104f <phase_2+0x19>
  # and bomb should explode
  40104a:	e8 b0 03 00 00       	callq  4013ff <explode_bomb>
```

proceeding from there, phase 2 sets up a loop to process the numbers on the stack.
looking at the following two commands, it seems 2 separate index variables are created:

```objdump
  # create array index, i, starting at A[1]
  # int i = 1
  40104f:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  # int j = 1 ?? not as clear what this will be used for yet...
  401054:	bd 01 00 00 00       	mov    $0x1,%ebp
```

moving on, then we see we temporarily store `j` in a new register (let's call it `k`), then add the value from the array element _before_ our current one to it:

```objdump
  # copies 1 to eax (return value)
  # int k = j
  401059:	89 e8                	mov    %ebp,%eax
  # adds value @ mem addr rbx - 4 to value at reg eax
  # k += A[i - 1]
  40105b:	03 43 fc             	add    -0x4(%rbx),%eax
```

finally, we check if our temporary value `k` is now equal to our current array element (`A[i]`).
if not, explode the bomb. 
otherwise, move on.

```objdump
  # cond = k == A[i]
  40105e:	39 03                	cmp    %eax,(%rbx)
  # if not cond
  401060:	74 05                	je     401067 <phase_2+0x31>
  # then boom
  401062:	e8 98 03 00 00       	callq  4013ff <explode_bomb>
```

ere at the end of the loop, we increment i & j (our array index & our strange number), then check our loop condition (j < 6) & continue accordingly:

```objdump
  # otherwise continue
  # j++
  401067:	83 c5 01             	add    $0x1,%ebp
  # i++
  40106a:	48 83 c3 04          	add    $0x4,%rbx
  # if j is 6, end loop
  40106e:	83 fd 06             	cmp    $0x6,%ebp
  # otherwise iterate again from beginning of loop
  401071:	75 e6                	jne    401059 <phase_2+0x23>
```

in the end, we should have checked that for each a[i] in A, a[i] == a[i - 1] + j, meaning in c, this was a loop that likely looked something like this:

```c
void phase_2(char* input) {
    int i, j = 1;
    
    while (j<6) {
        if (input[i - 1] + j != input[i]) {
            explode_bomb();
        }
        i++;
        j++;
    }
}
```

phase 3
-------

right away, this phase makes room for 18 bytes on the stack & grabs stores addresses for this stack frame + `0x8` & `0xc` in `%rcx` & `%rdx`.

```objdump
  401145:	48 83 ec 18          	sub    $0x18,%rsp
  401149:	48 8d 4c 24 08       	lea    0x8(%rsp),%rcx
  40114e:	48 8d 54 24 0c       	lea    0xc(%rsp),%rdx
```

next, we see it uses `scanf` to read something from input, passing in a format string from `%esi`, set here:

```objdump
  401153:	be e9 24 40 00       	mov    $0x4024e9,%esi
```

which we can examine as a string using the following gdb command:

```
(gdb) p *(char**)0x4024e9
$xx = 0x4024e9 "%d %d"
```

this tells us `scanf` is going to attempt to extract 2 decimal numbers from the input, storing them on the stack, starting at `%rsp`.

checking the stack after returning from `scanf` shows that we loaded our values from "1 2 " to `%rsp + 12` & `%rsp + 8` respectively:

```
(gdb) p *(int *)($rsp + 8)
$60 = 2
(gdb) p *(int *)($rsp + 12)
$61 = 1
```

also worth noting, `%rdx` & `%rcx` both go overwritten during `scanf`:

```
# before
18: /d *(int *)$rdx = 32767
17: /x $rdx = 0x7fffffffe22c
16: /d *(int *)$rcx = -7368
15: /x $rcx = 0x7fffffffe228

#after
18: /d *(int *)$rdx = <error: Cannot access memory at address 0x0>
17: /x $rdx = 0x0
16: /d *(int *)$rcx = <error: Cannot access memory at address 0x20>
15: /x $rcx = 0x20
```

after reading the 2 numbers from input, the first test that results in an explosion on failure is encountered, requiring that more than 1 number was read from the input string:

```objdump
  401162:	83 f8 01             	cmp    $0x1,%eax
  401165:	7f 05                	jg     40116c <phase_3+0x27>
  401167:	e8 93 02 00 00       	callq  4013ff <explode_bomb>
  40116c:	...
```

assuming we pass that test, then we go straight into another comparison (also sending us to `explode_bomb` if we fail), this time using `cmpl` & `ja`:

```objdump
  40116c:	83 7c 24 0c 07       	cmpl   $0x7,0xc(%rsp)
  401171:	77 66                	ja     4011d9 <phase_3+0x94>
```

as cmpl is somewhat hard to find documentation on, some searching around found the following on a lecture slide from princeton's spring 2016 offering of cos217 ([slide 61][1]):

> Example: subl src, dest
>  • Compute sum (dest+(-src))
>  • Assign sum to dest
>  • ZF: set to 1 iff sum == 0
>  • SF: set to 1 iff sum < 0
>  • CF: set to 1 iff unsigned overflow
>    • Set to 1 iff dest<src
>  • OF: set to 1 iff signed overflow
>    • Set to 1 iff
>      ```
>      (dest>0 && src<0 && sum<0) ||
>      (dest<0 && src>0 && sum>=0)
>      ```
>
>  Example: cmpl src, dest
>  • Same as subl
>  • But does not affect dest 

given `ja` jumps if the flags `CF` & `ZF` are both set to 0, this means that we are doing 2 things:

1. subtracting 7 from the value at the memory address 0xc (12) bytes above the stack pointer
2. then setting ZF to 1 if the result is 0 & setting CF to 1 if there's any unsigned overflow, meaning that our value was <= 7
3. finally this all means that if our value was greater than 7, then we explode

given our value at `%rsp + 12` is 1, we don't get vaporized this time:

```
20: /d *(int *)($rsp + 12) = 1

```

assuming we survived so far, then we move our value @ 12 bytes above `%rsp` to `%eax`, then jump... somewhere?

```objdump
  401177:	ff 24 c5 00 24 40 00 	jmpq   *0x402400(,%rax,8)
```

breaking that hairy jump down, it's calculated in a few parts:

1. `*0x402400`: a deref'd memory address
2. `(,%rax,8)`: a calculation of values `(a,b,c)` that seems a little opaque so far

the first one's easy to get: `(gdb) p/x *0x402400` gives us `0x4011a8`, which is then used as an offset from the address calculated in the 2nd part.

that second part looks harder, but if we ignore it for a moment, we'll see our choices for it are even more limited.
looking ahead, there's another requirement to have this number be less than 5:

```objdump
  4011e3:	83 7c 24 0c 05       	cmpl   $0x5,0xc(%rsp)
  4011e8:	7f 06                	jg     4011f0 <phase_3+0xab>
```

picking 5 as it will pass both checks on our first number so far, we see it will first jump us to an instruction setting `%eax` to 0, then again to the middle of a series of arithmetic operations:

```objdump
  40118c:	b8 00 00 00 00       	mov    $0x0,%eax
  401191:	eb 35                	jmp    4011c8 <phase_3+0x83>
...
  4011c8:	2d 7c 01 00 00       	sub    $0x17c,%eax
  4011cd:	05 7c 01 00 00       	add    $0x17c,%eax
  4011d2:	2d 7c 01 00 00       	sub    $0x17c,%eax
```

ending with a value of -380 for `%eax`. because the final comparison requires this be our 2nd input number, we now have a solution:

```
5 -380
```

references
==========
 
[1]: https://www.cs.princeton.edu/courses/archive/spr16/cos217/lectures/14_Assembly2.pdf
