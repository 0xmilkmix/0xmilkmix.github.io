---
layout: post
title: "Angr management: first steps and limitations"
---

# Introduction
Last summer I took some time to finally learn about Z3 as I was solving some crackme (see [Using Z3 to solve crackme](https://speakerdeck.com/milkmix/using-z3-to-solve-crackme)) but in order to stay true to my hipster reputation I had to try something cooler this year: [angr](https://github.com/angr/angr). This tool has already been used numerous times during CTF but rarely with a detailed explanation of what was happening. The documentation is really well written and reading the examples will help you while running into most problems you might face.

I will explain some concepts of angr and its basic usage through examples in this post. All samples and scripts from this post can be found on my [github](https://github.com/0xmilkmix/sandbox/tree/master/angr).

# Symbolic execution and constraints solvers
I will not go into details what symbolic execution and constraints solvers are, two of the main pillars of angr but rather just give a short intro.

Basically symbolic execution means that contrary to when you run your code on your CPU and have concrete values in the registers, memory, etc. here you will have a symbolic representation of the values. When the program hits a branch both paths can then be analyzed (note: this could lead to a path explosion problem on large programs). This means that your program states can be represented as formulas and as such, reaching a specific code path is translated to solving the formulas that brought to it.

Constraints solvers on the other hand are some magical algorithms. When given a problem it can tell you if it is solvable and if it is, provide you with solutions. You must be aware that they give you solutions, NOT the solution. Best known constraint solver is Z3.

Combining the two is awesome as it allows you to know what are the required inputs to reach a given execution point. Several projects are built on this, including angr, BAP, BitBlaze, SAGE, Mayhem, [S2E](http://s2e.epfl.ch) or [KLEE](http://klee.github.io). KLEE is a bit different as it does not directly works on binaries but on LLVM bytecode as well as make some KLEE API calls in the code to analyse. As such it is required to pair it with [mcsema](https://github.com/trailofbits/mcsema) (or equivalent) in order to generate IR from compiled binary (TBH I haven't tried this yet but it is on the list, also TrailOfBits recently [announced](https://blog.trailofbits.com/2016/08/02/engineering-solutions-to-hard-program-analysis-problems/) that they are pairing McSema with PySymEmu and not KLEE). There are also commercial solutions build on those concepts such as Reven.

For more details about both I encourage you to consult Jonathan [slides](http://shell-storm.org/talks/SecurityDay2015_dynamic_symbolic_execution_Jonathan_Salwan.pdf) which are really complete on the subject.

# Angr

## Intro
Angr is a project from the Computer Security Lab at UC Santa Barbara and sponsored by DARPA. The project's goal is to develop a framework allowing to create binary analysis tools on top of it without having to redevelop core concepts.

## Terminology
Angr project is composed of smaller elements including the following:

- Loader and core
- Symbolic execution engine
  - simuvex : the semantic representation based on top of VEX
  - paths : higher level representation of basic blocks executed since the initial state
  - path groups : organize paths in stashes
- Solver engine
  - claripy
  - design allows to use any constraint solver
  - currently only implements z3
- ...

# First steps
Let's use the following C code as a first example:

**Note:** I'm using source code for learning purpose only. Angr is working directly on assembly code.

``` c
// simple.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

static void    invalid(void)
{
  printf("invalid input\n");
  exit(1);
}

int            main(void)
{
  char         buff[32];
  unsigned int n;

  if (!(n = read(0, buff, 32)))
    invalid();
  buff[n - 1] = 0;
  if (strlen(buff) < 5)
    invalid();
  for (n = 0; buff[n]; ++n)
  {
    if ((buff[n] < 0x61) || (0x7a < buff[n]))
      invalid();
  }
  for (n = 0; n < (strlen(buff) - 1); ++n)
  {
    if (buff[n] != (buff[n+1] + 1))
      invalid();
  }
  printf("yeah you got it!\n");
  return 0;
}
```

In this case the program is only taking its inputs from *stdin* and it is fairly straightforward to see that we need to reach the last *printf* and avoid the calls to *invalid()*.
Inputs coming from stdin are automatically handled by angr using its symbolic implementation of parts of the libc as such here we only need to let it set the initial state automatically and tell the explorer what we want to find and what we want to avoid. Internally angr will group paths in stashes:

- active : the current path
- deadended : problematic path that ended up into invalid instructions, no valid states to reach its successors, ...
- found : paths matching the *find* condition set in the explorer
- unconstrained : paths where the instruction pointer is controlled by user's input
- ...

In our case we just have to care about the *found* stash. The paths that we want to avoid can be given either as single address or as a list of addresses. This results in the following python script and output.

``` python
# solve_simple.py
# ~10s on 3GHz core i7
import angr

b = angr.Project('./simple')
pg = b.factory.path_group()
pg.explore(find = 0x00400785, avoid = 0x00400686)
print pg
if len(pg.found) == 0:
  exit(1)
found = pg.found[0]
flag = found.state.posix.dumps(0).strip('\0\n')
print flag
```

``` shell
$ ./solve_simple.py
<PathGroup with 74 avoid, 51 unsat, 18 active, 1 found>
hgfed
```

Easy right? Now let's say that we want the first character to be *z*. What we need is retrieve the initial state and add a constraint to it stating that the first byte from stdin is equal to a BitVectorValue (BVV) of *0x7a*.

``` python
# solve_simple.py
import angr

b = angr.Project('./simple')
initial_state = b.factory.entry_state(args = ['./simple'])
c = initial_state.se.state.posix.files[0].read_from(1)
initial_state.add_constraints(c == angr.claripy.BVV(0x7a, 8))
pg = b.factory.path_group(initial_state)
pg.explore(find = 0x00400785, avoid = 0x00400686)
print pg
if len(pg.found) == 0:
  exit(1)
found = pg.found[0]
flag = found.state.posix.dumps(0).strip('\0\n')
print flag
```

``` shell
$ ./solve_simple.py
<PathGroup with 74 avoid, 51 unsat, 18 active, 1 found>
zhgfed
```

# More complex scenario
Our second example will be a crackme I wrote for Insomni'hack 2013 qualifications: *chall*. By opening it up in *radare2* (street_creds++) and inspecting the main function we can collect the following information:

- input is expected as the first command line argument
- upon wrong input program exits with code 0x2a
- upon success *"flag is : ..."* is displayed and operation are made using the key to decode an URL
- 11th character should be *\**
- forgot to explicitly state it in the binary itself but 12th character should be *A* in order for the decoding to work and the argument should be 16 characters long

This time we should create a symbolic value to represent the first command line argument and we will add some constraints on it.

``` python
# solve_chall.py
import angr

b = angr.Project('./chall')
arg1 = angr.claripy.BVS('arg1', 16 * 8)
initial_state = b.factory.entry_state(args=['./chall', arg1])
for c in arg1.chop(8):
  initial_state.add_constraints(c != 0)
initial_state.add_constraints(arg1.get_byte(11) == angr.claripy.BVV(0x2a, 8))
initial_state.add_constraints(arg1.get_byte(12) == angr.claripy.BVV(0x41, 8))
initial_state.add_constraints(arg1.get_byte(15) == angr.claripy.BVV(0x99, 8))
```

**Note:** it is also possible to set the initial state at another instruction's address in the binary and set the memory, stack and registers accordingly. This method can be used when working on Windows binaries where the all system is not yet supported by angr.

We then retrieve the addresses of the blocks to avoid:

``` shell
[0x004007f0]> /c mov edi, 0x2a
0x00400b12   # 5: mov edi, 0x2a
0x00400b3c   # 5: mov edi, 0x2a
0x00400bbd   # 5: mov edi, 0x2a
0x00400c05   # 5: mov edi, 0x2a
[0x004007f0]> pi 2 @@hit*
mov edi, 0x2a
call sym.imp.exit
mov edi, 0x2a
call sym.imp.exit
mov edi, 0x2a
call sym.imp.exit
mov edi, 0x2a
call sym.imp.exit
```

and finish the script:

``` python
pg = b.factory.path_group(initial_state, threads = 8)
pg.explore(find=0x00400c0f, avoid=[0x00400c05, 0x00400bbd, 0x00400b3c, 0x00400b12, 0x00400ad3])
print pg
if len(pg.found) == 0:
  exit(1)
found = pg.found[0]
flag = found.state.se.any_str(arg1)
print binascii.hexlify(flag)
```

``` shell
$ ./solve_chall.py
...
```

Aaaannnnd it never finishes (well at least on my laptop where I only let it run max 3 hours). Time to investigate the problem using the debugging feature:

``` python
# add this line after the project initialization
angr.path_group.l.setlevel('DEBUG')
```

``` shell
$ ipython ./solve_chall.py
...
DEBUG   | 2016-07-26 11:37:33,441 | angr.path_group | Round 68: stepping <PathGroup with 14 avoid, 1 active>
<ctrl+c>
In [11]: print pg.active
[<Path with 68 runs (at 0x400935)>]
```

The trick, as I learned on #angr, is to run the script in *ipython* in order to be able to debug the active path group. Here we can see that the code called a C library (z3 backend) from claripy and it seems to be z3 that never returns. The problematic basic block is in the *calc_offset* function as we can see from the active path group.

**Note:** be patient after sending the SIGINT to the python process

After some reversing we can see that *calc_offset* is always returning the same value (ok, I cheated here). Since we don't want angr to loose itself in this and we know that the return value is always *137*, we can create a hook to replace the call.

Hooks in angr are a way to replace instructions. It should not be confused with tools such as [Detours](https://www.microsoft.com/en-us/research/project/detours/) that are actually hooking a function. When setting a hook the number of bytes to overwrite should be given (ex: 5 for a call instruction on x86). If the value is 0 then the initial instruction is not replaced but the hook gets called before it.

``` python
# solve_chall.py
# ~5sec on 3GHz core i7
def calc_offset(state):
  state.regs.rax = state.se.BVV(137, 64)

b = angr.Project('./chall')
arg1 = angr.claripy.BVS('arg1', 16 * 8)
b.hook(0x00400bdc, calc_offset, length = 5)
...
```

``` shell
$ ./solve_chall.py
<PathGroup with 15 avoid, 1 found>
696e736f6d6e696861636b2a410b1699
```

This time it worked!

# Exploitation
An interesting note from the introduction is that one path group's stash is named unconstrained and contains paths where the instruction pointer is controlled by external inputs (ex: user’s ones). This is related to vulnerabilities leading to code execution using shellcode or code reuse :)

``` c
// bof_01.c
// Note: -fno-stack-protector -z execstack

#include <stdio.h>
#include <string.h>
#include <unistd.h>

static void     vulnerable(void)
{
  char          buffer[128];

  read(0, buffer, 150);
  printf("user input: %s\n", buffer);
  return;
}

int main(int argc, char** argv)
{
    vulnerable();
    return 0;
}
```

By default angr does not save unconstrained paths so we have to explicitly tell it to do so while setting up the path group. We will then iterate through all path groups until angr founds an unconstrained one. This is made possible in our case as angr implements the *read()* function and as such knows that up to *nbyte* might taint the memory at *buf*. Upon the symbolic execution of the *ret* instruction in *vulnerable*, angr knows that execution paths lead to an overwrite of the saved EIP.

We can easily see that the destination buffer of the *read* call is under the user’s control as pictured in the *ipython* example below.

In order to check exploitability, we need to validate that the instruction pointer is fully under our control (ie. fully symbolic). It is also possible to add a constraint on the found path so that EIP is equal to 0x41414141 before retrieving a faulty input :)

``` python
# solve_buf_01.py
# ~2sec on 3GHz core i7
# inspired from angr insomnihack_aeg example
import angr
import binascii

def check_continuity(address, addresses, length):
  for i in range(length):
    if not address + i in addresses:
      return False
  return True

def find_symbolic_buffer(state, length):
  stdin_file = state.posix.get_file(0)
  sym_addrs = [ ]
  for var in stdin_file.variables():
    sym_addrs.extend(state.memory.addrs_for_name(var))
  for addr in sym_addrs:
    if check_continuity(addr, sym_addrs, length):
      yield addr

def fully_symbolic(state, variable):
  for i in range(state.arch.bits):
    if not state.se.symbolic(variable[i]):
      return False
  return True

p = angr.Project('./bof_01')
extras = {so.REVERSE_MEMORY_NAME_MAP, so.TRACK_ACTION_HISTORY}
# required to be able to call addrs_for_name() and retrieve addresses
es = p.factory.entry_state(add_options = extras)
pg = p.factory.path_group(es, save_unconstrained = True, threads = 4)

ep = None
while ep is None:
  pg.step()
  if len(pg.unconstrained) > 0:
    print "found some unconstrained paths, checking exploitability"
    for u in pg.unconstrained:
      if fully_symbolic(u.state, u.state.regs.pc):
        ep = u
        break
    pg.drop(stash = 'unconstrained')
print "found a path which looks exploitable"
ep.state.add_constraints(ep.state.regs.pc == ep.state.se.BVV(0x41414141, 32))
buff = ep.state.posix.dumps(0).rstrip(chr(0))
print "should overflow with input of length: %i" % len(buff)
print "proceed with pwning using:"
print "python -c \"import binascii; print binascii.unhexlify(\'%s\')\"" % binascii.hexlify(buff)
```

``` shell
$ ipython -i ./solve_bof_01.py
found some unconstrained paths, checking exploitability
found a path which looks exploitable
should overflow with input of length: 144
proceed with pwning using:
python -c "import binascii; print binascii.unhexlify('808040808080808080808080808080808080808080808080800808080808080808080808080808080808080808080808080808080808080808080000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041414141')"

In [10]: addrs = find_symbolic_buffer(ep.state, 16)
print ep.state.memory.load(addrs.next, 8)
<BV64 file_/dev/stdin_0_0_3_1200[15:8] .. file_/dev/stdin_0_0_3_1200[23:16] .. ...
```

Of course it is also possible to add constraint to the input , as performed in the first example, in order to prevent null bytes or forbidden characters if needed.

I will stop here, but it is possible to retrieve the address of the user's input and as such generate an exploit. The full process is called Automatic Exploit Generation (AEG) and is one of the goals at DARPA CGC.

# Conclusion
This post should give you an overview of the possibilities of angr and honestly, I'm still learning while doing my experimentations. I also think that limiting it to crackme is not exploiting the full potential of such a framework. The developers already presented a paper on using selective symbolic execution to enhance code coverage and bug hunting while fuzzing: [Driller](https://www.internetsociety.org/sites/default/files/blogs-media/driller-augmenting-fuzzing-through-selective-symbolic-execution.pdf). Some other projects published recently, like this IOCTL [resolver](http://thunderco.re/project/security/2016/07/18/fuzzing-ioctls/).

Such solutions are facing one major problem: scalability. It gives fairly good results on CTF binaries or dedicated functions, however it does not scale easily to larger binaries. This could be ok when analyzing smaller binaries on embedded devices but not larger ones. Angr being implemented in python, speed is not on his side but I guess that the ShellPhish team is using a distributed version to compete in the DARPA Cyber Grand Challenge and have better performances.
