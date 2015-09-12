---
layout: post
title: "FreeBSD/amd64 でのプログラム実行"
date: 2014-11-30 11:48:04 +0900
comments: true
categories: freebsd
---

	% uname -a
	FreeBSD freebsd11 11.0-CURRENT FreeBSD 11.0-CURRENT #3 r272236: Sat Nov  1 17:27:44 JST 2014     root@freebsd11:/usr/obj/usr/src/sys/GENERIC  amd64

プログラムの実行は execve システムコールから開始する。sys\_execve() から kern\_execve()、do\_execve() の順に呼ばれ、do\_execve() の中にある

``` c kern_exec.c
	for (i = 0; error == -1 && execsw[i]; ++i) {
		if (execsw[i]->ex_imgact == NULL ||
		    execsw[i]->ex_imgact == img_first) {
			continue;
		}
		error = (*execsw[i]->ex_imgact)(imgp);
	}
```

の部分でプログラム起動用の処理を呼ぶ。image activator のリストというのがこの execsw で、execsw[i]->ex\_imgact に各実行形式に応じた関数ポインタが入っている。シェルスクリプトなら exec\_shell\_imageact()、64bit の ELF なら exec\_elf64\_imgact といったように。これらの関数は実行形式にマッチしない場合に -1 を返すので、for 文の条件を満たし、順に exescw を見ていく。また、execsw は NULL 終端になっている。これらの処理は各フォーマット毎に kern/imgact\_xxx.c (xxx はフォーマットの種類) という名前のソースに記述されている。

以降、interpreter と ELF(64bit) のそれぞれに対して image activator が何をしているのかを調べる。

## interpreter の場合

次のようなシェルスクリプトを例にの実行開始処理を見て行く。

``` sh echo.sh
#!/bin/sh -v

echo "hoge"
```

前述のように開始処理は exec\_shell\_imageact() で行われる。まずは "#!" から始まっているかをチェック。

``` c imgact_shell.c
	if (((const short *)image_header)[0] != SHELLMAGIC)
		return (-1);

	imgp->interpreted |= IMGACT_SHELL;
```

IMGACT\_SHELL は再帰的に interpreter を呼ばれているかを知るためのもの。

``` c imgact_shell.c
	maxp = &image_header[MIN(vattr.va_size, MAXSHELLCMDLEN)];
	ihp = &image_header[2];
```
maxp にスクリブトの最終位置を、iph に "#!" の次を指すようにする。

``` c imgact_shell.c
	/*
	 * Find the beginning and end of the interpreter_name.  If the
	 * line does not include any interpreter, or if the name which
	 * was found is too long, we bail out.
	 */
	while (ihp < maxp && ((*ihp == ' ') || (*ihp == '\t')))
		ihp++;
	interpb = ihp;
	while (ihp < maxp && ((*ihp != ' ') && (*ihp != '\t') && (*ihp != '\n')
	    && (*ihp != '\0')))
		ihp++;
	interpe = ihp;
```

コメントによると上のコードにおける interpb の 'b' は "beginning"、interpe の 'e' は "end"。というわけで、interpb から interpe にかけてが "#!" に続くインタプリタ名、今回は "/bin/sh" になる。

次はインタプリタに渡すオプション。ここでは optb/obpte が interpb/interpe に対応する。というか、このあたりはコメントが詳しいので何をやっているのかが分かりやすい。

``` c imgact_shell.c
	/*
	 * Find the beginning of the options (if any), and the end-of-line.
	 * Then trim the trailing blanks off the value.  Note that some
	 * other operating systems do *not* trim the trailing whitespace...
	 */
	while (ihp < maxp && ((*ihp == ' ') || (*ihp == '\t')))
		ihp++;
	optb = ihp;
	while (ihp < maxp && ((*ihp != '\n') && (*ihp != '\0')))
		ihp++;
	opte = ihp;
	if (opte == maxp)
		return (ENOEXEC);
	while (--ihp > optb && ((*ihp == ' ') || (*ihp == '\t')))
		opte = ihp;
```

最後にそのインタプリタが実行するスクリプトのファイル名を知る必要がある。これは exec\_shell\_imageact() の引数 imgp 経由で取得可能。imgp->args は struct image\_args へのポインタで、この構造体は /usr/src/sys/sys/imgact.h で定義されている。

``` c imgact.h
struct image_args {
        char *buf;              /* pointer to string buffer */
        char *begin_argv;       /* beginning of argv in buf */
        char *begin_envv;       /* beginning of envv in buf */
        char *endp;             /* current `end' pointer of arg & env strings */
        char *fname;            /* pointer to filename of executable (system space) */
        char *fname_buf;        /* pointer to optional malloc(M_TEMP) buffer */
        int stringspace;        /* space left in arg & env buffer */
        int argc;               /* count of argument strings */
        int envc;               /* count of environment strings */
        int fd;                 /* file descriptor of the executable */
};
```

これも丁寧にコメントが付いており、imgp->args->fname でファイル名がとれると分かる。

ここまでで インタプリタ(/usr/bin/sh)、そのオプション(-v)およびスクリプトファイル名(./echo.sh)が得られたことになる。これから先はこれらを組み合わせて "/usr/bin/sh -v ./echo.sh" というコマンドを作成する。作成先は imgp->args->begin\_argv。現状では "./echo.sh" となっている。

まずは各種長さの計算。

``` c imgact_shell.c
	offset = interpe - interpb + 1;			/* interpreter */
	if (opte > optb)				/* options (if any) */
		offset += opte - optb + 1;
	offset += strlen(fname) + 1;			/* fname of script */
	length = (imgp->args->argc == 0) ? 0 :
	    strlen(imgp->args->begin_argv) + 1;		/* bytes to delete */
```

offset にはインタプリタ、オプション、スクリプト名の合計を、length には元の begin\_argv から削除する長さを入れる。削除した分は offset だけ空けて詰めなければならないので、bcopy() で begin\_arg から length 先をコピーする。bcopy() の引数は順に src, dst, cnt で、src から dst に cnt だけコピーする。

``` c imgact_shell.c
	bcopy(imgp->args->begin_argv + length, imgp->args->begin_argv + offset,
	    imgp->args->endp - (imgp->args->begin_argv + length));
```

length だけ前に詰めたのだから、関係する個所からその分を調整しなくてはならない。

``` c imgact_shell.c
	offset -= length;		/* calculate actual adjustment */
	imgp->args->begin_envv += offset;
	imgp->args->endp += offset;
	imgp->args->stringspace -= offset;
```

これで begin\_argv に必要な余白を確保できた。後はその余白を埋めてコマンドを作成するだけ。delimiter は '\0'。もちろん argc の調整も忘れずに。

コマンドを作成したら、interpreter\_name にインタプリタ名を入れて return する。do\_execve() return にしたら、今度は elf64 の image activator で実行する。

## ELF (64bit) の場合

ELF フォーマットに関しては FreeBSD の場合

	man 5 elf

で調べられる。ネット上でも [Wikipedia](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format) をはじめ色々な所に説明があるので、ここでその詳細には触れない。

interpreter の場合同様、こちらも単純なプログラムを用いて処理を見ていく。

``` c exec_elf.c
int
main(int argc, char** argv)
{
        return 0;
}
```

64bit ELF の image activator は exec\_elf64\_imgact() なのだが、この名前はマクロで生成され、ソース上では

	__CONCAT(exec_, __elfN(imgact))

と記される。簡単に予想がつくだろうが、\_\_elfN は "elf" の後にワードサイズ、'\_'、引数(今回は "imgact") をつなげるマクロ。

まずはこのファイルが本当に実行可能な ELF なのかを確める。そのためにはファイル先頭のマジックナンバーが ELF のものであり、実行可能(ET\_EXEC)または shared object(ET\_DYN) であるかを見る。そうでない場合は次の image activator へ。上述のプログラムをコンパイルしたものは当然 ELF なのだが、ついでなので ELF ヘッダ 見てみると次のような情報が含まれていることが分かる。

	% readelf -h a.out
	ELF Header:
	  Magic:   7f 45 4c 46 02 01 01 09 00 00 00 00 00 00 00 00 
	  Class:                             ELF64
	  Data:                              2's complement, little endian
	  Version:                           1 (current)
	  OS/ABI:                            UNIX - FreeBSD
	  ABI Version:                       0
	  Type:                              EXEC (Executable file)
	  Machine:                           Advanced Micro Devices X86-64
	  Version:                           0x1
	  Entry point address:               0x4004a0
	  Start of program headers:          64 (bytes into file)
	  Start of section headers:          3232 (bytes into file)
	  Flags:                             0x0
	  Size of this header:               64 (bytes)
	  Size of program headers:           56 (bytes)
	  Number of program headers:         8
	  Size of section headers:           64 (bytes)
	  Number of section headers:         28
	  Section header string table index: 25


``` c imgact_elf.c
        if (__elfN(check_header)(hdr) != 0 ||
            (hdr->e_type != ET_EXEC && hdr->e_type != ET_DYN))
                return (-1);
```

実行可能な ELF であることがはっきりしたら program header table を解析する。

``` c imgact_elf.c
        phdr = (const Elf_Phdr *)(imgp->image_header + hdr->e_phoff);
        if (!aligned(phdr, Elf_Addr))
                return (ENOEXEC);
        n = 0;
        baddr = 0;
        for (i = 0; i < hdr->e_phnum; i++) {
                switch (phdr[i].p_type) {
                case PT_LOAD:
                        if (n == 0)
                                baddr = phdr[i].p_vaddr;
                        n++;
                        break;
                case PT_INTERP:
                        /* Path to interpreter */
                        if (phdr[i].p_filesz > MAXPATHLEN ||
                            phdr[i].p_offset > PAGE_SIZE ||
                            phdr[i].p_filesz > PAGE_SIZE - phdr[i].p_offset)
                                return (ENOEXEC);
                        interp = imgp->image_header + phdr[i].p_offset;
                        interp_name_len = phdr[i].p_filesz;
                        break;
                case PT_GNU_STACK:
                        if (__elfN(nxstack))
                                imgp->stack_prot =
                                    __elfN(trans_prot)(phdr[i].p_flags);
                        break;
                }
        }
```

program header table にはセグメントの情報が入っているが、ここで扱うのは 3 種類のみ。

- PT\_LOAD は loadable なセグメント、最初に現れた PT\_LOAD セグメントの virtual memory のアドレスを baddr に、PT\_LOAD セクションの数を n に入れる。

- PT\_INTERP はインタプリタを示すセグメントで、interp にインタプリタのパス名、interp\_name\_len にその長さを入れる。

- PT\_GNU\_STACK は以前 [FreeBSD/amd64 のシステムコール](/blog/2014/09/27/freebsd11-on-kvm) を調べた時にもわずかに顔を見せていたが、executable stack の存在を示すもので、PT\_GNU\_STACK がない場合には read/write/exec 可と見なし、ある場合にはそのフラグに従う。\_\_elfN(nxstack) は amd64 の場合常に 1、\_\_elfN(trans\_prot)(phdr[i].p\_flags) は ELF のフラグから virtual memory のフラグに変換する関数。

今回の例では program header table はこんな感じ。

	% readelf -lW a.out

	Elf file type is EXEC (Executable file)
	Entry point 0x4004a0
	There are 8 program headers, starting at offset 64

	Program Headers:
	  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
	  PHDR           0x000040 0x0000000000400040 0x0000000000400040 0x0001c0 0x0001c0 R E 0x8
	  INTERP         0x000200 0x0000000000400200 0x0000000000400200 0x000015 0x000015 R   0x1
	      [Requesting program interpreter: /libexec/ld-elf.so.1]
	  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x000804 0x000804 R E 0x200000
	  LOAD           0x000808 0x0000000000600808 0x0000000000600808 0x0001e4 0x0001f0 RW  0x200000
	  DYNAMIC        0x000830 0x0000000000600830 0x0000000000600830 0x000170 0x000170 RW  0x8
	  NOTE           0x000218 0x0000000000400218 0x0000000000400218 0x000030 0x000030 R   0x4
	  GNU_EH_FRAME   0x000758 0x0000000000400758 0x0000000000400758 0x000024 0x000024 R   0x4
	  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x8

	 Section to Segment mapping:
	  Segment Sections...
	   00     
	   01     .interp 
	   02     .interp .note.tag .hash .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
	   03     .ctors .dtors .jcr .dynamic .got.plt .data .bss 
	   04     .dynamic 
	   05     .note.tag 
	   06     .eh_frame_hdr 
	   07     

というわけで、program header table を読んだ後では

- baddr: 0x400000
- n: 2
- interp: "/libexec/ld-elf.so.1"
- interp\_name\_len: 21
- imgp->stack\_prot: 6 (RW)

となっており、readelf の結果と一致する。なお、GNU\_STACK の行を見ると stack は exec 不可だと分かる。もし exec 可にしたければ

	% clang -Wl,-z,execstack exec_elf.c

のように linker にオプションを渡してやれば良い。その場合

	  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RWE 0x8

と E フラグが付き、imgp->stack\_prot の値は 7 となる。

上記の readelf の結果からでは interpreter が分からないが、INTERP 型のセグメントは .inpterp セクションから成っているので、objdump を用いてセクションを表示させると、

	% objdump -s -j .interp a.out

	a.out:     file format elf64-x86-64-freebsd

	Contents of section .interp:
	 400200 2f6c6962 65786563 2f6c642d 656c662e  /libexec/ld-elf.
	 400210 736f2e31 00                          so.1.

で、こちらも想定通り。

続いて ELF に対応する brand を検索する。ELF と一言で言っても実際には OS の ABI や ターゲットの instruction set architecture によって内容が異なるため、elf\_brand\_list から現在実行しようとしている ELF に適した brand の情報を検索する。検索の仕方は \_\_elfN(get\_brandinfo) 内にコメントで記述されている通り。

	/*
     * We support four types of branding -- (1) the ELF EI_OSABI field
     * that SCO added to the ELF spec, (2) FreeBSD 3.x's traditional string
     * branding w/in the ELF header, (3) path of the `interp_path'
     * field, and (4) the ".note.ABI-tag" ELF section.
     */

ソースを見ると実際には (4) -> (1),(2) -> (3) の優先度になっており、今回の例では (4) の手法で brand を見付けていた。試しに

	% objcopy -R .note.tag a.out 

として .note.tag セクションを削除してから実行してみると、(1) の手法で見付けるようになる。

必要な brand 情報が手に入ったらいよいよ load の処理を行う。

``` c imgact_elf.c
        for (i = 0; i < hdr->e_phnum; i++) {
                switch (phdr[i].p_type) {
                case PT_LOAD:   /* Loadable segment */
                        if (phdr[i].p_memsz == 0)
                                break;
                        prot = __elfN(trans_prot)(phdr[i].p_flags);
                        error = __elfN(load_section)(imgp, phdr[i].p_offset,
                            (caddr_t)(uintptr_t)phdr[i].p_vaddr + et_dyn_addr,
                            phdr[i].p_memsz, phdr[i].p_filesz, prot,
                            sv->sv_pagesize);
                        if (error != 0)
                                return (error);

                        /*
                         * If this segment contains the program headers,
                         * remember their virtual address for the AT_PHDR
                         * aux entry. Static binaries don't usually include
                         * a PT_PHDR entry.
                         */
                        if (phdr[i].p_offset == 0 &&
                            hdr->e_phoff + hdr->e_phnum * hdr->e_phentsize
                                <= phdr[i].p_filesz)
                                proghdr = phdr[i].p_vaddr + hdr->e_phoff +
                                    et_dyn_addr;

                        seg_addr = trunc_page(phdr[i].p_vaddr + et_dyn_addr);
                        seg_size = round_page(phdr[i].p_memsz +
                            phdr[i].p_vaddr + et_dyn_addr - seg_addr);

                        /*
                         * Make the largest executable segment the official
                         * text segment and all others data.
                         *
                         * Note that obreak() assumes that data_addr + 
                         * data_size == end of data load area, and the ELF
                         * file format expects segments to be sorted by
                         * address.  If multiple data segments exist, the
                         * last one will be used.
                         */

                        if (phdr[i].p_flags & PF_X && text_size < seg_size) {
                                text_size = seg_size;
                                text_addr = seg_addr;
                        } else {
                                data_size = seg_size;
                                data_addr = seg_addr;
                        }
                        total_size += seg_size;
                        break;
                case PT_PHDR:   /* Program header table info */
                        proghdr = phdr[i].p_vaddr + et_dyn_addr;
                        break;
                default:
                        break;
                }
        }
```

loadable なセグメントに対しての case 内で \_\_elfN(load\_section) がセクションをロードする本体。セグメント単位でロードしているのに load\_section という名前になっている理由は良く分からない。virtual memory 上にロードするので、ページ単位での扱いになる。この時、ページサイズに合わせる目的で使われるのが trunc\_page\_ps と round\_page\_ps。前者は切り捨て、後者は切り上げてページサイズに合わせる。

``` c imgact_elf.c
#define trunc_page_ps(va, ps)   ((va) & ~(ps - 1))
#define round_page_ps(va, ps)   (((va) + (ps - 1)) & ~(ps - 1))
```
virtual memory に関しては "The Design and Implementation of the FreeBSD Operating System Second Edition" の Chapter6 の主題なので、ここで詳しく見ることはしないが、大雑把に言うと \_\_elfN(load\_section) でやっているのはファイル上の offset と vitual memory 上のアドレスの map を作成し、ユーザの側からもアクセスするデータをユーザ空間に copyout している。

セグメントのロード後は text\_size、text\_addr、data\_size、data\_addr を計算する。コメントに書かれている通り、text\_size、text\_addr は実行可能な(PF\_X がある)セグメントのうち、最大サイズのものを使用する。data\_size、data\_addr はそれ以外のもので最後に現れたもの。全セグメントの合計サイズは total\_size に入れる。

サイズが分かったら、それが制限に引っかからないかを調べる。

``` c imgact_elf.c start:906
        PROC_LOCK(imgp->proc);
        if (data_size > lim_cur(imgp->proc, RLIMIT_DATA) ||
            text_size > maxtsiz ||
            total_size > lim_cur(imgp->proc, RLIMIT_VMEM) ||
            racct_set(imgp->proc, RACCT_DATA, data_size) != 0 ||
            racct_set(imgp->proc, RACCT_VMEM, total_size) != 0) {
                PROC_UNLOCK(imgp->proc);
                return (ENOMEM);
        }
```

data\_size, text\_size, total\_size などをチェックしている。text\_size は loader.conf に記述した ken.maxtsiz の値を、data\_size, total\_size は setrlimit で使定した値を使用する。ちなみに RLIMIT\_VMEM は RLIMIT\_AS と等価で、setrlimit の man では RLIMIT_AS の方を用いている。

racct\_set() は resource accounting の関数で、プロセスが使用するリソースの割り当てを変更する。

このようにしてサイズが確定したらそれを基に virtual memory の text, data の位置とサイズを設定する。

``` c imgact_elf.c start:916
        vmspace = imgp->proc->p_vmspace;
        vmspace->vm_tsize = text_size >> PAGE_SHIFT;
        vmspace->vm_taddr = (caddr_t)(uintptr_t)text_addr;
        vmspace->vm_dsize = data_size >> PAGE_SHIFT;
        vmspace->vm_daddr = (caddr_t)(uintptr_t)data_addr;

        /*
         * We load the dynamic linker where a userland call
         * to mmap(0, ...) would put it.  The rationale behind this
         * calculation is that it leaves room for the heap to grow to
         * its maximum allowed size.
         */
        addr = round_page((vm_offset_t)vmspace->vm_daddr + lim_max(imgp->proc,
            RLIMIT_DATA));
        PROC_UNLOCK(imgp->proc);

        imgp->entry_addr = entry;
```

addr には data セグメントの次に来るページの先頭アドレスが入る。RLIMIT\_DATA は sbrk() で拡張できる最大を示すものであるため、ヒープも含まれることから、``The Desgin and Implementation of the FreeBSD Operating System Second Edition'' の p.70 Figure 3.3 によると addr が指すのは shared libraries の領域になる。

``` c imgact_elf.c start:934
        if (interp != NULL) {
                int have_interp = FALSE;
                VOP_UNLOCK(imgp->vp, 0);
                if (brand_info->emul_path != NULL &&
                    brand_info->emul_path[0] != '\0') {
                        path = malloc(MAXPATHLEN, M_TEMP, M_WAITOK);
                        snprintf(path, MAXPATHLEN, "%s%s",
                            brand_info->emul_path, interp);
                        error = __elfN(load_file)(imgp->proc, path, &addr,
                            &imgp->entry_addr, sv->sv_pagesize);
                        free(path, M_TEMP);
                        if (error == 0)
                                have_interp = TRUE;
                }
                if (!have_interp && newinterp != NULL) {
                        error = __elfN(load_file)(imgp->proc, newinterp, &addr,
                            &imgp->entry_addr, sv->sv_pagesize);
                        if (error == 0)
                                have_interp = TRUE;
                }
                if (!have_interp) {
                        error = __elfN(load_file)(imgp->proc, interp, &addr,
                            &imgp->entry_addr, sv->sv_pagesize);
                }
                vn_lock(imgp->vp, LK_EXCLUSIVE | LK_RETRY);
                if (error != 0) {
                        uprintf("ELF interpreter %s not found\n", interp);
                        return (error);
                }
        } else
                addr = et_dyn_addr;
```

interp は前述の通り "/libexec/ld-elf.so.1" と NULL ではないので、if 文の中に入る。brand\_info->emul\_path も newinterp も brand\_info 由来の値で、elf64 ではともに NULL であったことから、今回は

``` c imgact_elf.c start:955
                        error = __elfN(load_file)(imgp->proc, interp, &addr,
                            &imgp->entry_addr, sv->sv_pagesize);
```

が実行される。なお、NULL でない場合、emul\_path は interp の起点となる path、newinterp は interp に代わって使用する interpreter 名として使われる。

\_\_elfN(load_file)() は

``` c imgact_elf.c start:584
/*
 * Load the file "file" into memory.  It may be either a shared object
 * or an executable.
 *
 * The "addr" reference parameter is in/out.  On entry, it specifies
 * the address where a shared object should be loaded.  If the file is
 * an executable, this value is ignored.  On exit, "addr" specifies
 * where the file was actually loaded.
 *
 * The "entry" reference parameter is out only.  On exit, it specifies
 * the entry point for the loaded file.
 */
static int
__elfN(load_file)(struct proc *p, const char *file, u_long *addr,
        u_long *entry, size_t pagesize)
```

とある通り、file を addr に、つまりは libexec/ld-elf.so.1 を addr にロードし、実際にロードされたアドレスで addr の値を上書きする。imgp->entry_addr には libexec/ld-elf.so.1 の entry point が入る。

ようやく exec\_elf64\_imgact() も終わりが見えて来た。最後の処理は

``` c imgact_elf.c start:966
        /*
         * Construct auxargs table (used by the fixup routine)
         */
        elf_auxargs = malloc(sizeof(Elf_Auxargs), M_TEMP, M_WAITOK);
        elf_auxargs->execfd = -1;
        elf_auxargs->phdr = proghdr;
        elf_auxargs->phent = hdr->e_phentsize;
        elf_auxargs->phnum = hdr->e_phnum;
        elf_auxargs->pagesz = PAGE_SIZE;
        elf_auxargs->base = addr;
        elf_auxargs->flags = 0;
        elf_auxargs->entry = entry;

        imgp->auxargs = elf_auxargs;
        imgp->interpreted = 0;
        imgp->reloc_base = addr;
        imgp->proc->p_osrel = osrel;
```

と program header や libexec/ld-elf.so.1 をロードしたアドレスを imgp に入れるだけ。

と、libexec/ld-elf.so.1 をロードしただけでは目的のプログラムはまだ実行できそうにない。そもそも libexec/ld-elf.so.1 が何なのかというと、これは

	man ld-elf.so.1

とすれば分かるが、shared object リンカ、ローダであり、実のところカーネルの仕事はこの ld-elf.so.1 を実行するまでになる。

では、ld-elf.so.1 をどうやって実行するのかというと、exec\_elf64\_imgact() の呼出し元である do\_execve() を見る必要がある。

exec\_elf64\_imgact() の最後で imgp->interpreted は 0 になっているため、return 直後の if 文には入らず、まずは

``` c kern_exec.c start:558
        /*
         * Do the best to calculate the full path to the image file.
         */
        if (imgp->auxargs != NULL &&
            ((args->fname != NULL && args->fname[0] == '/') ||
             vn_fullpath(td, imgp->vp, &imgp->execpath, &imgp->freepath) != 0))
                imgp->execpath = args->fname;
```

が実行される。ここではコメントにある通り、実行する image file(a.out とか) の full path を計算する。'/' から始まっていた場合には元々 full path であったということなので、 vn\_fullpath() は呼ばれずに直接 imgp->execpath に args->fname を入れる。そうでなければ vn\_fullpath() で full path を求める。

続いて stack よりも下位アトレスにある諸々の準備。

``` c kern_exec.c start:575
        /*
         * Copy out strings (args and env) and initialize stack base
         */
        if (p->p_sysent->sv_copyout_strings)
                stack_base = (*p->p_sysent->sv_copyout_strings)(imgp);
        else
                stack_base = exec_copyout_strings(imgp);
```

p->p\_sysent->sv\_copyout\_strings や p->p\_sysent->sv\_fixup が NULL でない場合には custom のルーチンがあるので、そちらを使う。NULL の場合には default のものを使う。今回、p->p\_sysent->sv\_copyout\_strings は NULL ではなかったものの、そのアドレスは exec\_copyout\_strings() のものだった。exec\_copyout\_strings() は関数の先頭にあるコメントに

``` c kern_exec.c start:1225
/*
 * Copy strings out to the new process address space, constructing new arg
 * and env vector tables. Return a pointer to the base so that it can be used
 * as the initial stack pointer.
 */
register_t *
exec_copyout_strings(imgp)
        struct image_params *imgp;
```

とある。再び ``The Desgin and Implementation of the FreeBSD Operating System Second Edition'' の p.70 Figure 3.3 を参照すると、ps\_string struct, signal code, env strings, argv strings, env pointers argv pointers あたりをコピーするらしい。「らしい」というのはコードを見ると、どうも少し異っているように思えるからで、

``` c kern_exec.c start:1261
        destp = (uintptr_t)arginfo;

        /*
         * install sigcode
         */
        if (szsigcode != 0) {
                destp -= szsigcode;
                destp = rounddown2(destp, sizeof(void *));
                copyout(p->p_sysent->sv_sigcode, (void *)destp, szsigcode);
        }

        /*
         * Copy the image path for the rtld.
         */
        if (execpath_len != 0) {
                destp -= execpath_len;
                imgp->execpathp = destp;
                copyout(imgp->execpath, (void *)destp, execpath_len);
        }

        /*
         * Prepare the canary for SSP.
         */
        arc4rand(canary, sizeof(canary), 0);
        destp -= sizeof(canary);
        imgp->canary = destp;
        copyout(canary, (void *)destp, sizeof(canary));
        imgp->canarylen = sizeof(canary);
```

destp がコピー先のアドレス。これを見ると、signal code をコピーした次は rtld(run-time link-editor) 用に image の path を、さらに SSP(stack smashing protection) の canary として乱数をコピーしている。

他にも pagesizes array なんかもコピーしていて、その次にようやく strings の出番となる。

``` c kern_exec.c start:1335
        stringp = imgp->args->begin_argv;
        argc = imgp->args->argc;
        envc = imgp->args->envc;

        /*
         * Copy out strings - arguments and environment.
         */
        copyout(stringp, (void *)destp, ARG_MAX - imgp->args->stringspace);
```

ここまで来たら、Figure 3.3 のレイアウト通りになっている。ps_string struct のメンバに値を入れつつ env pointers、argv pointers も初期化する。初期化の順番は argv が先なので注意。

これで exec\_copyout\_strings() は終わり。戻ってきたら


``` c kern_exec.c start:583
        /*
         * If custom stack fixup routine present for this process
         * let it do the stack setup.
         * Else stuff argument count as first item on stack
         */
        if (p->p_sysent->sv_fixup != NULL)
                (*p->p_sysent->sv_fixup)(&stack_base, imgp);
        else
                suword(--stack_base, imgp->args->argc);
```

argc のコピーなんかを行う。

この先には signal や setuid/setgid なんかの処理があるのだが、これらはもっと先の Chapter で扱う内容なので、ここでは触れない。

ようやく ld-elf.so.1 を実行するところまで来た。

``` c kern_exec.c start:837
        /* Set values passed into the program in registers. */
        if (p->p_sysent->sv_setregs)
                (*p->p_sysent->sv_setregs)(td, imgp, 
                    (u_long)(uintptr_t)stack_base);
        else
                exec_setregs(td, imgp, (u_long)(uintptr_t)stack_base);
```

例によって custom な setregs を使えるようにしてある。

exec\_setregs() はアーキテクチャに依存している。その処理の中で

``` c sys/amd64/amd64/machdep.c start:955
        bzero((char *)regs, sizeof(struct trapframe));
        regs->tf_rip = imgp->entry_addr;
        regs->tf_rsp = ((stack - 8) & ~0xFul) + 8;
        regs->tf_rdi = stack;           /* argv */
        regs->tf_rflags = PSL_USER | (regs->tf_rflags & PSL_T);
        regs->tf_ss = _udatasel;
        regs->tf_cs = _ucodesel;
        regs->tf_ds = _udatasel;
        regs->tf_es = _udatasel;
        regs->tf_fs = _ufssel;
        regs->tf_gs = _ugssel;
        regs->tf_flags = TF_HASSEGS;
        td->td_retval[1] = 0;
```
		
というふうに execve 呼出し元の thread 構造体のレジスタを初期化している。instruction register である RIP には imgp->entry\_addr を代入しているが、これは image activator 内で ld-elf.so.1 をロードした時にそのエントリポイントが割り当てられている。よって execve をしたスレッドのコンテキスト復帰時には ld-elf.so.1 の実行が開始される。

ld-elf.so.1 が何をしているかは[『リンカ・ローダ実践開発テクニック』](http://www.amazon.co.jp/%E3%83%AA%E3%83%B3%E3%82%AB%E3%83%BB%E3%83%AD%E3%83%BC%E3%83%80%E5%AE%9F%E8%B7%B5%E9%96%8B%E7%99%BA%E3%83%86%E3%82%AF%E3%83%8B%E3%83%83%E3%82%AF%E2%80%95%E5%AE%9F%E8%A1%8C%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AB%E5%BF%85%E9%A0%88%E3%81%AE%E6%8A%80%E8%A1%93-COMPUTER-TECHNOLOGY-%E5%9D%82%E4%BA%95-%E5%BC%98%E4%BA%AE/dp/4789838072)等に記載がある。興味がある向きは読んでみると面白いと思う。この本は x86(32bit) 上の FreeBSD-4.10 での記述で、バージョンだけを見ると古く感じるかもしれないが、このあたりのことはそんなに劇的に変わっていないので今読んでも十分役に立つ。共有ライブラリ関連の記述が短か過ぎて分かりにくかったけれど。

それにしても長くなり過ぎた。もっと端折って書くべきだったかも。

