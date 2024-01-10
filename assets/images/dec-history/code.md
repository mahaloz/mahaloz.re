
#### Example Codes
##### Source
```c
// ====================
if (a && b)
  puts("first");

puts("second");
if (b)
  puts("third");

sleep(1);
puts("fourth");
```

##### IDA Pro*
```c
// ====================
if ( a && b )
{
  puts("first");
  puts("second");
  goto LABEL_6;
}
puts("second");
if ( b ) {
LABEL_6:
  puts("third");
}
sleep(1u);
puts("fourth");
```

##### DREAM 
```c
// ====================
if (a && b) {
    puts("first");
    puts("second");
}
if (!a || !b)
    puts("second");
if (b)
    puts("third");
sleep(0x1);
puts("fourth");
```

##### rev.ng
```c
if (!a) {
    puts("second");
    if (b)
        puts("third");
    sleep(0x1);
    puts("fourth");
} else if (b) {
    puts("first");
    puts("second");
    puts("third");
    sleep(0x1);
    puts("fourth");
} else {
    puts("second");
    if (b)
        puts("third");
    sleep(0x1);
    puts("fourth");
}
```

