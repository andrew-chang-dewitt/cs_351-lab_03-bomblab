setup:

objdump -d bomb.c > bomb.asm
note fn names for phases & explode_bomb seems important

start gdb on ./bomb
setup breakpoints for all 6 phases + secret phase & another on explode_bomb as safeguard to prevent explosions
run w/ no args until first breakpoint on first phase

examine bomb.asm @ phase_1
note call to compare strings for equality, w/ first arg being program input & 2nd arg being supplied by program

si until @ strings_not_equal, then
print (char*)$rdi (user input)
print (char*)$rsi (program string)

save output of program string as first line in ./inputs.txt
comment out phase_1 breakpoint as it should be solved now

// break here for now, killing debug session & resuming later w/ following commands:
gdb bomb
source breakpoints.txt
run < inputs.txt
