# Plugins

## IO plugins

All access to files, network, debugger, etc. is wrapped by an IO abstraction layer that allows radare to treat all data as if it were just a file.

IO plugins are the ones used to wrap the open, read, write and 'system' on virtual file systems. You can make radare understand anything as a plain file. E.g., a socket connection, a remote radare session, a file, a process, a device, a gdb session, etc..

So, when radare reads a block of bytes, it is the task of an IO plugin to get these bytes from any place and put them into internal buffer. An IO plugin is chosen by a file's URI to be opened. Some examples:

* Debugging URIs

    $ r2 dbg:///bin/ls
    $ r2 pid://1927

* Remote sessions

    $ r2 rap://:1234
    $ r2 rap://<host>:1234//bin/ls

* Virtual buffers

    $ r2 malloc://512
    shortcut for
    $ r2 -

You can get a list of the radare IO plugins by typing `radare2 -L`:

    $ r2 -L
    rw_  zip         Open zip files apk://foo.apk//MANIFEST or zip://foo.apk//theclass/fun.class, show files with: zip://foo.apk/, open all files with zipall:// (BSD)
    rwd  windbg      Attach to a KD debugger (LGPL3)
    rw_  sparse      sparse buffer allocation (sparse://1024 sparse://) (LGPL3)
    rw_  shm         shared memory resources (shm://key) (LGPL3)
    rw_  self        read memory from myself using 'self://' (LGPL3)
    rw_  rap         radare network protocol (rap://:port rap://host:port/file) (LGPL3)
    rwd  ptrace      ptrace and /proc/pid/mem (if available) io (LGPL3)
    rw_  procpid     /proc/pid/mem io (LGPL3)
    rw_  mmap        open file using mmap:// (LGPL3)
    rw_  malloc      memory allocation (malloc://1024 hex://cd8090) (LGPL3)
    r__  mach        mach debug io (unsupported in this platform) (LGPL)
    rw_  ihex        Intel HEX file (ihex://eeproms.hex) (LGPL)
    rw_  http        http get (http://radare.org/) (LGPL3)
    rw_  gzip        read/write gzipped files (LGPL3)
    rwd  gdb         Attach to gdbserver, 'qemu -s', gdb://localhost:1234 (LGPL3)
    r_d  debug       Debug a program or pid. dbg:///bin/ls, dbg://1388 (LGPL3)
    rw_  bfdbg       BrainFuck Debugger (bfdbg://path/to/file) (LGPL3)

## Implementing a new architecture

radare2 splits the logic of a CPU into several modules. You should write more than one plugin to get full support for a specific arch. Let's see which are those:

* r_asm : assembler and disassembler
* r_anal : code analysis (opcode,type,esil,..)
* r_reg : registers
* r_syscall : system calls
* r_debug : debugger

The most basic feature you usually want to support from a specific architecture is the disassembler. You first need to read into a human readable form the bytes in there.

Bear in mind that plugins can be compiled static or dynamically, this means that the arch will be embedded inside the core libraries or it will distributed as a separated shared library.

To configure which plugins you want to compile use the `./configure-plugins` script which accepts the flags --shared and --static to specify them. You can also add it manually inside the `plugins.def.cfg` and then remove the `plugins.cfg` and run `./configure-plugins` again to update the `libr/config.mk` and `libr/config.h`.

You may find some examples of external plugins in the `radare2-capstone` and `radare2-extras` repositories:

1. https://github.com/radare/radare2-capstone
2. https://github.com/radare/radare2-extras

## Writing the r_asm plugin

The official way to make third-party plugins is to distribute them into a separate repository. This is a sample disasm plugin:

```Makefile
$ cd my-cpu
$ cat Makefile
NAME=mycpu
R2_PLUGIN_PATH=$(shell r2 -hh|grep LIBR_PLUGINS|awk '{print $$2}')
CFLAGS=-g -fPIC $(shell pkg-config --cflags r_asm)
LDFLAGS=-shared $(shell pkg-config --libs r_asm)
OBJS=$(NAME).o
SO_EXT=$(shell uname|grep -q Darwin && echo dylib || echo so)
LIB=$(NAME).$(SO_EXT)

all: $(LIB)

clean:
	rm -f $(LIB) $(OBJS)

$(LIB): $(OBJS)
	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJS) -o $(LIB)

install:
	cp -f $(NAME).$(SO_EXT) $(R2_PLUGIN_PATH)

uninstall:
	rm -f $(R2_PLUGIN_PATH)/$(NAME).$(SO_EXT)
```

```c
$ cat mycpu.c
/* example r_asm plugin by pancake at 2014 */

#include <r_asm.h>
#include <r_lib.h>

#define OPS 17

static const char *ops[OPS*2] = {
	"nop", NULL,
	"if", "r",
	"ifnot", "r",
	"add", "rr",
	"addi", "ri",
	"sub", "ri",
	"neg", "ri",
	"xor", "ri",
	"mov", "ri",
	"cmp", "rr",
	"load", "ri",
	"store", "ri",
	"shl", "ri",
	"br", "r",
	"bl", "r",
	"ret", NULL,
	"sys", "i"
};

//b for byte, l for length
static int disassemble (RAsm *a, RAsmOp *op, const ut8 *b, int l) {
	char arg[32];
        int idx = (b[0]&0xf)\*2;
	op->size = 2;
	if (idx>=(OPS*2)) {
		strcpy (op->buf_asm, "invalid");
		return -1;
	}
	strcpy (op->buf_asm, ops[idx]);
	if (ops[idx+1]) {
		const char \*p = ops[idx+1];
		arg[0] = 0;
		if (!strcmp (p, "rr")) {
			sprintf (arg, "r%d, r%d", b[1]>>4, b[1]&0xf);
		} else
		if (!strcmp (p, "i")) {
			sprintf (arg, "%d", (char)b[1]);
		} else
		if (!strcmp (p, "r")) {
			sprintf (arg, "r%d, r%d", b[1]>>4, b[1]&0xf);
		} else
		if (!strcmp (p, "ri")) {
			sprintf (arg, "r%d, %d", b[1]>>4, (char)b[1]&0xf);
		}
		if (*arg) {
			strcat (op->buf_asm, " ");
			strcat (op->buf_asm, arg);
		}
	}
	return op->size;
}

RAsmPlugin r_asm_plugin_mycpu = {
        .name = "mycpu",
        .arch = "mycpu",
        .license = "LGPL3",
        .bits = 32,
        .desc = "My CPU disassembler",
        .disassemble = &disassemble,
};

#ifndef CORELIB
struct r_lib_struct_t radare_plugin = {
        .type = R_LIB_TYPE_ASM,
        .data = &r_asm_plugin_mycpu
};
#endif
```

To build and install this plugin just type this:

```
$ make
$ sudo make install
```

## Testing the plugin

This plugin is used by rasm2 and r2. You can verify that the plugin is properly loaded with this command:
```
$ rasm2 -L | grep mycpu
_d  mycpu        My CPU disassembler  (LGPL3)
```

Let's open an empty file using the 'mycpu' arch and write some random code there.

```
$ r2 -
 -- I endians swap
[0x00000000]> e asm.arch=mycpu
[0x00000000]> woR
[0x00000000]> pd 10
           0x00000000    888e         mov r8, 14
           0x00000002    b2a5         ifnot r10, r5
           0x00000004    3f67         ret
           0x00000006    7ef6         bl r15, r6
           0x00000008    2701         xor r0, 1
           0x0000000a    9826         mov r2, 6
           0x0000000c    478d         xor r8, 13
           0x0000000e    6b6b         store r6, 11
           0x00000010    1382         add r8, r2
           0x00000012    7f15         ret
```
Yay! it works.. and the mandatory oneliner too!

```
r2 -nqamycpu -cwoR -cpd' 10' -
```

## Static plugins in core

Pushing a new architecture into the main branch of r2 requires to modify several files in order to make it fit into the way the rest of plugins are built.

List of affected files:

* `plugins.def.cfg` : add the asm.mycpu plugin name string in there
* `libr/asm/p/mycpu.mk` : build instructions
* `libr/asm/p/asm_mycpu.c` : implementation
* `libr/include/r_asm.h` : add the struct definition in there

Check out how the NIOS II cpu was implemented by reading those commits:

Implement RAsm plugin:
https://github.com/radare/radare2/commit/933dc0ef6ddfe44c88bbb261165bf8f8b531476b

Implement RAnal plugin:
https://github.com/radare/radare2/commit/ad430f0d52fbe933e0830c49ee607e9b0e4ac8f2

## Write a disassembler plugin with another programming language

* [Python](https://github.com/radare/radare2-bindings/blob/master/libr/lang/p/test-py-asm.py)
* [Javascript](https://github.com/radare/radare2-bindings/blob/master/libr/lang/p/dukasm.js)

More examples to come...

## Write a debugger plugin

* Adding the debugger registers profile into the shlr/gdb/src/core.c
* Adding the registers profile and architecture support in the libr/debug/p/debug_native.c and libr/debug/p/debug_gdb.c
* Add the code to apply the profiles into the function `r_debug_gdb_attach(RDebug *dbg, int pid)`

If you want to add support for the gdb, you can see the register profile in the active gdb session using command `maint print registers`.

## More to come..

The next logic step would be to implement and analysis plugin.

...

* Related article: http://radare.today/extending-r2-with-new-plugins/

Some commits related to "Implementing a new architecture"

* Extensa: https://github.com/radare/radare2/commit/6f1655c49160fe9a287020537afe0fb8049085d7
* Malbolge: https://github.com/radare/radare2/pull/579
* 6502: https://github.com/radare/radare2/pull/656
* h8300: https://github.com/radare/radare2/pull/664
* GBA: https://github.com/radare/radare2/pull/702
* CR16: https://github.com/radare/radare2/pull/721/ && https://github.com/radare/radare2/pull/726
* XCore: https://github.com/radare/radare2/commit/bb16d1737ca5a471142f16ccfa7d444d2713a54d
* Sharp LH5801: https://github.com/neuschaefer/radare2/commit/f4993cca634161ce6f82a64596fce45fe6b818e7
* MSP430: https://github.com/radare/radare2/pull/1426 
* HP PA-RISC: https://github.com/radare/radare2/commit/f8384feb6ba019b91229adb8fd6e0314b0656f7b
* V810: https://github.com/radare/radare2/pull/2899
* TMS320: https://github.com/radare/radare2/pull/596



## Implementing a new pseudo architecture

Example:

* **Z80**: https://github.com/radare/radare2/commit/8ff6a92f65331cf8ad74cd0f44a60c258b137a06


## Implementing a new analysis plugin

```makefile
NAME=anal_snes
R2_PLUGIN_PATH=$(shell r2 -hh|grep LIBR_PLUGINS|awk '{print $$2}')
CFLAGS=-g -fPIC $(shell pkg-config --cflags r_anal)
LDFLAGS=-shared $(shell pkg-config --libs r_anal)
OBJS=$(NAME).o
SO_EXT=$(shell uname|grep -q Darwin && echo dylib || echo so)
LIB=$(NAME).$(SO_EXT)

all: $(LIB)

clean:
	rm -f $(LIB) $(OBJS)

$(LIB): $(OBJS)
	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJS) -o $(LIB)

install:
	cp -f anal_snes.$(SO_EXT) $(R2_PLUGIN_PATH)

uninstall:
	rm -f $(R2_PLUGIN_PATH)/anal_snes.$(SO_EXT)
```

**anal_snes.c:**
```c
/* radare - LGPL - Copyright 2015 - condret */

#include <string.h>
#include <r_types.h>
#include <r_lib.h>
#include <r_asm.h>
#include <r_anal.h>
#include "snes_op_table.h"

static int snes_anop(RAnal *anal, RAnalOp *op, ut64 addr, const ut8 *data, int len) {
	memset (op, '\0', sizeof (RAnalOp));
	op->size = snes_op[data[0]].len;
	op->addr = addr;
	op->type = R_ANAL_OP_TYPE_UNK;
	switch (data[0]) {
		case 0xea:
			op->type = R_ANAL_OP_TYPE_NOP;
			break;
	}
	return op->size;
}

struct r_anal_plugin_t r_anal_plugin_snes = {
	.name = "snes",
	.desc = "SNES analysis plugin",
	.license = "LGPL3",
	.arch = R_SYS_ARCH_NONE,
	.bits = 16,
	.init = NULL,
	.fini = NULL,
	.op = &snes_anop,
	.set_reg_profile = NULL,
	.fingerprint_bb = NULL,
	.fingerprint_fcn = NULL,
	.diff_bb = NULL,
	.diff_fcn = NULL,
	.diff_eval = NULL
};

#ifndef CORELIB
struct r_lib_struct_t radare_plugin = {
	.type = R_LIB_TYPE_ANAL,
	.data = &r_anal_plugin_snes,
	.version = R2_VERSION
};
#endif
```
After compiling radare2 will list this plugin in the output:
```
_d__  _8_16      snes        LGPL3   SuperNES CPU
```

**snes_op_table**.h: https://github.com/radare/radare2/blob/master/libr/asm/arch/snes/snes_op_table.h

Example:

* **6502**: https://github.com/radare/radare2/commit/64636e9505f9ca8b408958d3c01ac8e3ce254a9b
* **SNES**: https://github.com/radare/radare2/commit/60d6e5a1b9d244c7085b22ae8985d00027624b49

## Implementing a new format

### To enable virtual addressing

In `info` add `et->has_va = 1;` and `ptr->srwx` with the `R_BIN_SCN_MAP;` attribute

### Create a folder with file format name in libr/bin/format

**Makefile:**

```Makefile
NAME=bin_nes
R2_PLUGIN_PATH=$(shell r2 -hh|grep LIBR_PLUGINS|awk '{print $$2}')
CFLAGS=-g -fPIC $(shell pkg-config --cflags r_bin)
LDFLAGS=-shared $(shell pkg-config --libs r_bin)
OBJS=$(NAME).o
SO_EXT=$(shell uname|grep -q Darwin && echo dylib || echo so)
LIB=$(NAME).$(SO_EXT)

all: $(LIB)

clean:
	rm -f $(LIB) $(OBJS)

$(LIB): $(OBJS)
	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJS) -o $(LIB)

install:
	cp -f $(NAME).$(SO_EXT) $(R2_PLUGIN_PATH)

uninstall:
	rm -f $(R2_PLUGIN_PATH)/$(NAME).$(SO_EXT)
 
```

**bin_nes.c:**

```c
#include <r_bin.h>

static int check(RBinFile *arch);
static int check_bytes(const ut8 *buf, ut64 length);

static void * load_bytes(RBinFile *arch, const ut8 *buf, ut64 sz, ut64 loadaddr, Sdb *sdb){
	check_bytes (buf, sz);
	return R_NOTNULL;
}


static int check(RBinFile *arch) {
	const ut8 \*bytes = arch ? r_buf_buffer (arch->buf) : NULL;
	ut64 sz = arch ? r_buf_size (arch->buf): 0;
	return check_bytes (bytes, sz);
}

static int check_bytes(const ut8 *buf, ut64 length) {
	if (!buf || length < 4) return false;
	return (!memcmp (buf, "\x4E\x45\x53\x1A", 4));
}


static RBinInfo* info(RBinFile *arch) {
	RBinInfo \*ret = R_NEW0 (RBinInfo);
	if (!ret) return NULL;

	if (!arch || !arch->buf) {
		free (ret);
		return NULL;
	}
	ret->file = strdup (arch->file);
	ret->type = strdup ("ROM");
	ret->machine = strdup ("Nintendo NES");
	ret->os = strdup ("nes");
	ret->arch = strdup ("6502");
	ret->bits = 8;

	return ret;
}


struct r_bin_plugin_t r_bin_plugin_nes = {
	.name = "nes",
	.desc = "NES",
	.license = "BSD",
	.init = NULL,
	.fini = NULL,
	.get_sdb = NULL,
	.load = NULL,
	.load_bytes = &load_bytes,
	.check = &check,
	.baddr = NULL,
	.check_bytes = &check_bytes,
	.entries = NULL,
	.sections = NULL,
	.info = &info,
};

#ifndef CORELIB
struct r_lib_struct_t radare_plugin = {
	.type = R_LIB_TYPE_BIN,
	.data = &r_bin_plugin_nes,
	.version = R2_VERSION
};
#endif

```


### Some Examples

* XBE - https://github.com/radare/radare2/pull/972
* COFF - https://github.com/radare/radare2/pull/645
* TE - https://github.com/radare/radare2/pull/61
* Zimgz - https://github.com/radare/radare2/commit/d1351cf836df3e2e63043a6dc728e880316f00eb
* OMF - https://github.com/radare/radare2/commit/44fd8b2555a0446ea759901a94c06f20566bbc40