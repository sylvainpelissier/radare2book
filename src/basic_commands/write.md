## Writing Data

Radare can manipulate a loaded binary file in many ways. You can resize the file, move and copy/paste bytes, insert new bytes (shifting data to the end of the block or file), or simply overwrite bytes. New data may be given as a wide-string, assembler instructions, or the data may be read in from another file.

Resize the file using the `r` command. It accepts a numeric argument. A positive value sets a new size for the file. A negative one will truncate the file to the current seek position minus N bytes.

```
r 1024      ; resize the file to 1024 bytes
r -10 @ 33  ; strip 10 bytes at offset 33
```
Write bytes using the `w` command. It accepts multiple input formats like inline assembly, endian-friendly dwords, files, hexpair files, wide strings:

```
w?
Usage: w[x] [str] [<file] [<<EOF] [@addr]
| w[1248][+-][n]       increment/decrement byte,word..
| w foobar             write string 'foobar'
| w0 [len]             write 'len' bytes with value 0x00
| w6[de] base64/hex    write base64 [d]ecoded or [e]ncoded string
| wa[?] push ebp       write opcode, separated by ';' (use '"' around the command)
| waf f.asm            assemble file and write bytes
| waF f.asm            assemble file and write bytes and show 'wx' op with hexpair bytes of assembled code
| wao[?] op            modify opcode (change conditional of jump. nop, etc)
| wA[?] r 0            alter/modify opcode at current seek (see wA?)
| wb 010203            fill current block with cyclic hexpairs
| wB[-]0xVALUE         set or unset bits with given value
| wc[?][jir+-*?]       write cache list/undo/commit/reset (io.cache)
| wd [off] [n]         duplicate N bytes from offset at current seek (memcpy) (see y?)
| we[?] [nNsxX] [arg]  extend write operations (insert instead of replace)
| wf[fs] -|file        write contents of file at current offset
| wh r2                whereis/which shell command
| wm f0ff              set binary mask hexpair to be used as cyclic write mask
| wo[?] hex            write in block with operation. 'wo?' fmi
| wp[?] -|file         apply radare patch file. See wp? fmi
| wr 10                write 10 random bytes
| ws pstring           write 1 byte for length and then the string
| wt[?] file [sz]      write to file (from current seek, blocksize or sz bytes)
| ww foobar            write wide string 'f\x00o\x00o\x00b\x00a\x00r\x00'
| wx[?][fs] 9090       write two intel nops (from wxfile or wxseek)
| wv[?] eip+34         write 32-64 bit value honoring cfg.bigendian
| wz string            write zero terminated string (like w + \x00)
```

Some examples:

```
 [0x00000000]> wx 123456 @ 0x8048300
 [0x00000000]> wv 0x8048123 @ 0x8049100
 [0x00000000]> wa jmp 0x8048320
```

### Write Over

The `wo` command (write over) has many subcommands, each combines the existing data with the new data using
an operator. The command is applied to the current block. Supported operators include XOR, ADD, SUB...

```
[0x000003fc]> wo?
Usage: wo[asmdxoArl24]   [hexpairs] @ addr[!bsize]
| wo[24aAdlmorwx]               without hexpair values, clipboard is used
| wo2 [val]                     2=  2 byte endian swap (word)
| wo4 [val]                     4=  4 byte endian swap (dword)
| wo8 [val]                     8=  8 byte endian swap (qword)
| woa [val]                     +=  addition (f.ex: woa 0102)
| woA [val]                     &=  and
| wod [val]                     /=  divide
| woD[algo] [key] [IV]          decrypt current block with given algo and key
| woe [from to] [step] [wsz=1]  ..  create sequence
| woE [algo] [key] [IV]         encrypt current block with given algo and key
| woi                           inverse bytes in current block
| wol [val]                     <<= shift left
| wom [val]                     *=  multiply
| woo [val]                     |=  or
| wop[DO] [arg]                 De Bruijn Patterns
| wor [val]                     >>= shift right
| woR                           random bytes (alias for 'wr $b')
| wos [val]                     -=  substraction
| wow [val]                     ==  write looped value (alias for 'wb')
| wox [val]                     ^=  xor  (f.ex: wox 0x90)
```

It is possible to implement basic algorithms using radare core primitives and `wo`. A sample session performing xor(90) + add(01, 02):

```
[0x7fcd6a891630]> px
- offset -       0 1  2 3  4 5  6 7  8 9  A B  C D  E F
0x7fcd6a891630  4889 e7e8 6839 0000 4989 c48b 05ef 1622
0x7fcd6a891640  005a 488d 24c4 29c2 5248 89d6 4989 e548
0x7fcd6a891650  83e4 f048 8b3d 061a 2200 498d 4cd5 1049
0x7fcd6a891660  8d55 0831 ede8 06e2 0000 488d 15cf e600
[0x7fcd6a891630]> wox 90
[0x7fcd6a891630]> px
- offset -       0 1  2 3  4 5  6 7  8 9  A B  C D  E F
0x7fcd6a891630  d819 7778 d919 541b 90ca d81d c2d8 1946
0x7fcd6a891640  1374 60d8 b290 d91d 1dc5 98a1 9090 d81d
0x7fcd6a891650  90dc 197c 9f8f 1490 d81d 95d9 9f8f 1490
0x7fcd6a891660  13d7 9491 9f8f 1490 13ff 9491 9f8f 1490
[0x7fcd6a891630]> woa 01 02
[0x7fcd6a891630]> px
- offset -       0 1  2 3  4 5  6 7  8 9  A B  C D  E F
0x7fcd6a891630  d91b 787a 91cc d91f 1476 61da 1ec7 99a3
0x7fcd6a891640  91de 1a7e d91f 96db 14d9 9593 1401 9593
0x7fcd6a891650  c4da 1a6d e89a d959 9192 9159 1cb1 d959
0x7fcd6a891660  9192 79cb 81da 1652 81da 1456 a252 7c77
```

Is is also possible to decrypt data and write it over the previous encrypted data. The full list of supported algorithms is given by `woE?`. The first parameter is the key and the second the IV:
```
[0x00000000]> b 32
[0x00000000]> px
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x00000000  66d8 184d e86e 0085 d56c 9afa 1cc9 e659  f..M.n...l.....Y
0x00000010  dbd4 0456 1106 9cc3 02da 5e7b e43d 01a8  ...V......^{.=..
[0x00000000]> woD aes-cbc 00112233445566778899aabbccddeeff 000102030405060708090a0b0c0d0e0f
Written 32 byte(s)
[0x00000000]> px
- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
0x00000000  4865 6c6c 6f20 776f 726c 6420 2100 0000  Hello world !...
0x00000010  0000 0000 0000 0000 0000 0000 0000 0000  ................
[0x00000000]> 
```
