---
layout: post
title: c++ 异常处理（下）
description: c++ 异常处理（下）
category: C++
tags: C++ Exception-handling
refer_author: Ming Dong
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

前面一篇博文简单介绍了c++异常处理的流程，但在一些细节上一带而过了，比如，\_Unwind\_RaiseException是怎样重建函数现场的，personality routine是怎样清理栈上变量的等，这些细节涉及到很多与语言层面无关的东西，本文尝试介绍一下这些细节的具体实现。

相关的数据结构
--------------

如前所述，unwind的进行需要编译器生成一定的数据来支持，这些数据保存了与每个可能抛异常的函数相关的信息以供运行时查找，那么，编译器都保存了哪些信息呢？根据Itanium
ABI的[定义](http://www.intel.com/content/dam/www/public/us/en/documents/guides/itanium-software-runtime-architecture-guide.pdf)，主要包括以下三类：

1. unwind table，这个表记录了与函数相关的信息，共三个字段：函数的起始地址，函数的结束地址，一个info
block指针。
1. unwind descriptor table，
这个列表用于描述函数中需要unwind的区域的相关信息。
1. 语言相关的数据(language specific data area)，用于上层语言内部的处理。

以上数据结构的描述来自Itanium ABI的标准定义，但在具体实现时，这些数据是怎么组织以及放到了哪里则是由编译器来决定的，对于GCC来说，所有与unwind相关的数据都放到
了.eh\_frame及.gcc\_except\_table这两个section里面了，而且它的格式与内容和标准的定义稍稍有些不同。

.eh\_frame区域
----------------

.eh\_frame的格式与.debug\_frame是很相似的（不完全相同），属于[DWARF](http://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-Core-generic/LSB-Core-generic/normativerefs.html#STD.DWARF3)标准中的一部分。所有由GCC编译生成的需要支持异常处理的程序都包含了DWARF格式的数据与字节码，这些数据与字节码的主要作用有两个：

1. 描述函数调用栈的结构（layout）

2. 异常发生后，指导unwinder怎么进行unwind。

DWARF的字节码功能很强
大，它是图灵完备的，这意味着仅仅通过DWARF就可以做几乎任何事情。但是从数据的组织上来看，DWARF实在略显复杂晦涩，因此很少有人愿意去碰，本
文也只是简单介绍其中与异常处理相关的东西。本质上来说，eh\_frame像是一张表，它用于描述怎样根据程序中某一条指令来设置相应的寄存器，从而返回
到当前函数的调用函数中去，它的作用可以用如下表格来形象地描述。

<table class="confluenceTable"><tbody>
<tr>
<td class="confluenceTd"> program counter </td>
<td class="confluenceTd"> CFA </td>
<td class="confluenceTd"> ebp </td>
<td class="confluenceTd"> ebx  </td>
<td class="confluenceTd"> eax </td>
<td class="confluenceTd"> return address  </td>
</tr>
<tr>
<td class="confluenceTd"> 0xfff0003001 </td>
<td class="confluenceTd"> rsp+32 </td>
<td class="confluenceTd"> \*(cfa-16) </td>
<td class="confluenceTd">\*(cfa-24)  </td>
<td class="confluenceTd"> eax=edi </td>
<td class="confluenceTd"> *(cfa-8)  </td>
</tr>
<tr>
<td class="confluenceTd"> 0xfff0003002 </td>
<td class="confluenceTd"> rsp+32 </td>
<td class="confluenceTd"> \*(cfa-16) </td>
<td class="confluenceTd">  </td>
<td class="confluenceTd"> eax=edi </td>
<td class="confluenceTd"> *(cfa-8)  </td>
</tr>
<tr>
<td class="confluenceTd"> 0xfff0003003 </td>
<td class="confluenceTd"> rsp+32 </td>
<td class="confluenceTd"> \*(cfa-16) </td>
<td class="confluenceTd">\*(cfa-24)  </td>
<td class="confluenceTd"> eax=edi </td>
<td class="confluenceTd"> *(cfa-8)  </td>
</tr>
</tbody></table>


上表中，CFA(canonical frame address的缩写)表示一个基地址，用于作为当前函数中的其它地址的起始地址，使得其它地址可以用与该基地址的偏移来表示，由于这个表可能要覆盖很多程序指令，因此这个表的体积有可能是很大的，甚至比程序本身的代码量还要大。而在实际中，为了减少这个表的体积，GCC通常会对它进行压缩编码，以及尽可能减少要覆盖的指令的数量，比如，只对会抛异常的函数里的特定区域指令进行记录。

具体的实现上，eh\_frame由一个CIE (Common Information Entry) 及多个 FDE
(Frame Description Entry)组成，它们在内存中是连续存放的：

![](/assets/image/2014-05/2014-05-13-C-Plus-plus-Exception-Handle_files/021647204843602.png)

 CIE及FDE格式的定义可以参看如下：

 CIE结构: 

![](/assets/image/2014-05/2014-05-13-C-Plus-plus-Exception-Handle_files/table1.png)



 FDE结构: 

![](/assets/image/2014-05/2014-05-13-C-Plus-plus-Exception-Handle_files/table2.png)

 

注意其中标注红色的字段：

1. Initial Instructions，Call Frame Instructions
这两字段里放的就是所谓的DWARF字节码，比如：DW\_CFA\_def\_cfa R
OFF，表示通过寄存器R及位移OFF来计算CFA，其功能类似于前面的表格中第二列指明的内容。

2. PC begin，PC
range，这两个字段联合起来表示该FDE所能覆盖的指令的范围，eh\_frame中所有的FDE最后会按照pc
begin排序进行存放。

3. 如果CIE中的Augmentation String中包含有字母"P"，则相应的Augmentation
Data中包含有指向personality routine的指针。

4. 如果CIE中的Augmentation String中包含有有字母“L”，则FDE中Aumentation
Data包含有language specific data的指针。

对一个elf文件通过如下命令：readelf -Wwf
xxx，可以读取其中关于.eh\_frame的数据：


    The section .eh_frame contains:

    00000000 0000001c 00000000 CIE
      Version:               1
      Augmentation:          "zPL"
      Code alignment factor: 1
      Data alignment factor: -8
      Return address column: 16
      Augmentation data:     00 d8 09 40 00 00 00 00 00 00

      DW_CFA_def_cfa: r7 ofs 8   ##以下为字节码
      DW_CFA_offset: r16 at cfa-8

    00000020 0000002c 00000024 FDE cie=00000000 pc=00400ac8..00400bd8
      Augmentation data:     00 00 00 00 00 00 00 00
      #以下为字节码
      DW_CFA_advance_loc: 1 to 00400ac9
      DW_CFA_def_cfa_offset: 16
      DW_CFA_offset: r6 at cfa-16
      DW_CFA_advance_loc: 3 to 00400acc
      DW_CFA_def_cfa_reg: r6
      DW_CFA_nop
      DW_CFA_nop
      DW_CFA_nop



对于由GCC编译出来的程序 来说，CIE，
FDE是其在unwind过程中恢复现场时所依赖的全部东西，而且是完备的，这里所说的恢复现场指的是恢复调用当前函数的函数的现场，比如，func1调
用func2，然后我们可以在func2里通过查询CIE，FDE恢复func1的现场。

CIE，FDE存在于每一个需要处理异常的ELF文件中，当异常发生时，根据当前PC值调用dl\_iterate\_phdr()函数就可以把当前程序所加载的所有模块轮询一遍，从而找到该PC所在模块的eh\_frame。



    for (n = info->dlpi_phnum; --n >= 0; phdr++)
        {
          if (phdr->p_type == PT_LOAD)
          {
            _Unwind_Ptr vaddr = phdr->p_vaddr + load_base;
            if (data->pc >= vaddr && data->pc < vaddr + phdr->p_memsz)
              match = 1;
          }
          else if (phdr->p_type == PT_GNU_EH_FRAME)
            p_eh_frame_hdr = phdr;
          else if (phdr->p_type == PT_DYNAMIC)
            p_dynamic = phdr;
        }



找到eh\_frame也就找到CIE，找到了CIE也就可以去搜索相应的FDE。

找到FDE及CIE后，就可以从这两数据表中提取相关的信息，并执行DWARF
字节码，从而得到当前函数的调用函数的现场，参看如下用于重建函数帧的函数：



    static _Unwind_Reason_Code
    uw_frame_state_for (struct _Unwind_Context *context, _Unwind_FrameState *fs)
    {
      struct dwarf_fde *fde;
      struct dwarf_cie *cie;
      const unsigned char *aug, *insn, *end;

      memset (fs, 0, sizeof (*fs));
      context->args_size = 0;
      context->lsda = 0;

      // 根据context查找FDE。
      fde = _Unwind_Find_FDE (context->ra - 1, &context->bases);
      if (fde == NULL)
        {
          /* Couldn't find frame unwind info for this function.  Try a
         target-specific fallback mechanism.  This will necessarily
         not provide a personality routine or LSDA.  */
    #ifdef MD_FALLBACK_FRAME_STATE_FOR
          MD_FALLBACK_FRAME_STATE_FOR (context, fs, success);
          return _URC_END_OF_STACK;
        success:
          return _URC_NO_REASON;
    #else
          return _URC_END_OF_STACK;
    #endif
        }

      fs->pc = context->bases.func;

      // 获取对应的CIE.
      cie = get_cie (fde);

      // 提取出CIE中的信息，如personality routine的地址。
      insn = extract_cie_info (cie, context, fs);
      if (insn == NULL)
        /* CIE contained unknown augmentation.  */
        return _URC_FATAL_PHASE1_ERROR;

      /* First decode all the insns in the CIE.  */
      end = (unsigned char *) next_fde ((struct dwarf_fde *) cie);

      // 执行dwarf字节码，从而恢复相应的寄存器的值。
      execute_cfa_program (insn, end, context, fs);

     // 定位到fde的相关数据
      /* Locate augmentation for the fde.  */
      aug = (unsigned char *) fde + sizeof (*fde);
      aug += 2 * size_of_encoded_value (fs->fde_encoding);
      insn = NULL;
      if (fs->saw_z)
        {
          _Unwind_Word i;
          aug = read_uleb128 (aug, &i);
          insn = aug + i;
        }

      // 读取language specific data的指针
      if (fs->lsda_encoding != DW_EH_PE_omit)
        aug = read_encoded_value (context, fs->lsda_encoding, aug,
                      (_Unwind_Ptr *) &context->lsda);

      /* Then the insns in the FDE up to our target PC.  */
      if (insn == NULL)
        insn = aug;
      end = (unsigned char *) next_fde (fde);

      // 执行FDE中的字节码。
      execute_cfa_program (insn, end, context, fs);

      return _URC_NO_REASON;
    }



通过如上的操作，unwinder就已经把调用函数的现场给重建起来了，这些现场信息包括：


    struct _Unwind_Context
    {
      void *reg[DWARF_FRAME_REGISTERS+1];  //必要的寄存器。
        void *cfa; // canoniacl frame address, 前面提到过，基地址。
        void *ra;// 返回地址。
        void *lsda;// 该函数对应的language specific data,如果存在的话。
        struct dwarf_eh_bases bases;
      _Unwind_Word args_size;
    };


### 实现Personality routine 

Peronality routine 的作用主要有两个：

1. 检查当前函数是否有相应的catch语句。

2. 清理当前函数中的局部变量。

十分不巧，这两件事情仅仅依靠运行时也是没法完成的，必须依靠编译器在编译时建立起相关的数据进行协助。对GCC来说，这些与抛异常的函数具体相
关的信息全部放在.gcc\_except\_table区域里去了，这些信息会作为Itanium
ABI接口中所谓的language specific data在unwinder 与c++
ABI之间传递，根据前面的介绍，我们知道在FDE中保存有指向language specific
data的指针，因此unwinder在重建现场的时候就已经把这些数据读取了出来，c++的ABI只要调用
\_Unwind\_GetLanguageSpecificData()就可以得到指向该数据的指针。

关于GCC下language specific
data的格式，在网上几乎找不到什么权威的文档，我只在llvm的官网上找到一个相关的[链接](http://mentorembedded.github.io/cxx-abi/exceptions.pdf)，这个文档对gcc\_except\_table作了很详细的说明，我对比了一下GCC源码里的personality
routine的相关实现，发现两者还是有些许出入，因此本文接下来的介绍主要基于对GCC相关源码的个人解读。

 

下图来源于网络，展示了gcc\_except\_table及language specific data
的格式：

![](/assets/image/2014-05/2014-05-13-C-Plus-plus-Exception-Handle_files/062249043291941.jpg)

  

由上图所示，LSDA主要由一个表头，后面紧跟着三张表组成。

###### 1. LSDA Header：

该表头主要用来保存接下来三张表的相关信息，如编码，及表的位移等，该表头主要包含六个域：

1. Landing pad起始地址的编码方式，长度为一个字节。

2. landing pad
起始地址，这是可选的，只有当前面指明的编码方式不等于DW\_EH\_PE\_omit时，这个字段才存在，此时读取这个字段就需要根据前面指定的编码方式进行读取，长度不固定。
如果这个字段不存在，则landing pad的起始地址需要通过调用\_Unwind\_GetRegionStart（）来获得，得到其实就是当前模块加载的起始地址，这是最常见的形式。

3. type table的编码方式，长度为一个字节。

4. type table的位移，类型为unsigned
LEB128，这个字段是可选的，只有3）中编码方式不等于DW\_EH\_PE\_omit时，这个才存在。

5. call site table的编码方式，长度为一个字节。

6. call site table 的长度，一个unsigned LEB128的值。

###### 2. call site table
1. 可能会抛异常的指令的地址，该地址是距Landing
pad起始地址的偏移，编码方式由LSDA表头中第一个字段指明。

2. 可能抛异常的指令的区域长度，该字段与1）一起表示一系列连续的指令，编码方式与1）相同。

3. 用于处理上述指令的Landing
pad的位移，这个值如果为0则表示不存在相应的landing pad。

4. 指明要采取哪些action，这是一个unsigned
LEB128的值，该值减1后作为下标获取action table中相应记录。

call site table中的记录按第一个字段也就是指令起始地址进行排序存放，因此unwind的时候可以加快对该表的搜索，unwind时，如果当前pc的值不在
call site table覆盖的范围内的话，搜索就会返回，然后就调用std::terminate()结束程序，这通常来说是不正常的行为。

如果在call site table中有对应的处理，但landing pad的地址却是0的话，表明当前函数既不存在catch语句，也不需要清理局部变量，这是一种正常情况，unwinder应该继续向上unwind，而
如果landing pad不为0，则表明该函数中有catch语句，但是这些catch能否处理抛出的异常则还要结合action字段，到type table中去进一步加以判断：

1. 如果action字段为0，则表明当前函数没有catch语句，但有局部变量需要清理。

2. 如果action字段不为0，则表明当前函数中存在catch语句，又因为catch是可能存在多个的，怎么知道哪个能够catch当前的异常呢？因此需要去检查action table中的表项。

###### 3. Action table

action
table中每一条记录是一个二元组，表示一个catch语句所对应的异常，或者表示当前函数所允许抛出的异常(exception specification)，该列表每条记录包含两个字段：

1. filter type，这是一个unsigned LEB128的数值，用于指向type table中的记录，该值有可能是负数。

2. 指向下一个action table中的下一条记录，这是当函数中有多个catch或exception specification
有多个时，将各个action 记录链接起来。

###### 4. Type Table

type table中存放的是异常类型的指针:

    std::type_info* type_tables[];

这个表被分成两部分，一部分是各个catch所对应的异常的类型，另一部分是该函数允许抛出的异常类型：

    void func() throw(int, string)
    {
    }

type table中这两部分分别通过正负下标来进行索引：

![](/assets/image/2014-05/2014-05-13-C-Plus-plus-Exception-Handle_files/070326314977733.jpg)

有了如上这些数据，personality
routine只需要根据当前的pc值及当前的异常类型，不断在上述表中查找，最后就能找到当前函数是否有landing
pad，如果有则返回\_URC\_INSTALL\_CONTEXT，指示unwinder跳过去执行相应的代码。

### 什么是Landing pad

在前面一篇博文里，我们简单提到了Landing
pad：指的是能够catch当前异常的catch语句。这个说法其实不确切。

准确来说，landing pad指的是unwinder之外的“用户代码”：

1. 用于catch相应的exception，对于一个函数来说，如果该函数中有catch语句，且能够处理当前的异常，则该catch就是landing
pad

2. 如果当前函数没有catch或者catch不能处理当前exception，则意味着异常还要从当前函数继续往上抛，因而unwind当前函数时有可能要进行相应的清理，此时这些清理局部变量的代码就是landing
pad。

从名字上来看，顾名思议，landing
pad指的是程序的执行流程在进入当前函数后，最后要转到这里去，很恰当的描述。

当landing
pad是catch语句时，这个比较好理解，前面我们一直说清理局部变量的代码，这是什么意思呢？这些清理代码又放在哪里？

为了说明这个问题，我们看一下如下代码：


    #include <iostream>
    #include <stddef.h>
    using namespace std;

    class cs
    {
        public:

            explicit cs(int i) :i_(i) { cout << "cs constructor:" << i << endl; }
            ~cs() { cout << "cs destructor:" << i_ << endl; }

        private:

            int i_;
    };

    void test_func3()
    {
        cs c(33);
        cs c2(332);

        throw 3;

        cs c3(333);
        cout << "test func3" << endl;
    }

    void test_func3_2()
    {
        cs c(32);
        cs c2(322);

        test_func3();

        cs c3(323);

        test_func3();
    }

    void test_func2()
    {
        cs c(22);

        cout << "test func2" << endl;
        try
        {
            test_func3_2();

            cs c2(222);
        }
        catch (int)
        {
            cout << "catch 2" << endl;
        }
    }

    void test_func1()
    {
        cout << "test func1" << endl;
        try
        {
            test_func2();
        }
        catch (...)
        {
            cout << "catch 1" << endl;
        }
    }

    int main()
    {
        test_func1();
        return 0;
    }


对于函数test\_func3\_2()来说，当test\_func3()抛出异常后，在unwind的第二阶段，我们知道
test\_func3\_2()中的局部变量c及c2是需要清理的，而c3则不用，那么编译器是怎么生成代码来完成这件事情的呢？当异常发生时，运行时是没
有办法知道当前哪些变量是需要清理的，因为这个原因编译器在生成代码的时候，在函数的末尾设置了多个出口，使得当异常发生时，可以直接跳到某一段代码就能
清理相应的局部变量，我们看看test\_func3\_2()编译后生成的对应的汇编代码：


    void test_func3_2()
    {
      400ca4:    55                     push   %rbp
      400ca5:    48 89 e5               mov    %rsp,%rbp
      400ca8:    53                     push   %rbx
      400ca9:    48 83 ec 48            sub    $0x48,%rsp
        cs c(32);
      400cad:    48 8d 7d e0            lea    0xffffffffffffffe0(%rbp),%rdi
      400cb1:    be 20 00 00 00         mov    $0x20,%esi
      400cb6:    e8 9f 02 00 00         callq  400f5a <_ZN2csC1Ei>
        cs c2(322);
      400cbb:    48 8d 7d d0            lea    0xffffffffffffffd0(%rbp),%rdi
      400cbf:    be 42 01 00 00         mov    $0x142,%esi
      400cc4:    e8 91 02 00 00         callq  400f5a <_ZN2csC1Ei>

        test_func3();
      400cc9:    e8 5a ff ff ff         callq  400c28 <_Z10test_func3v>

        cs c3(323);
      400cce:    48 8d 7d c0            lea    0xffffffffffffffc0(%rbp),%rdi
      400cd2:    be 43 01 00 00         mov    $0x143,%esi
      400cd7:    e8 7e 02 00 00         callq  400f5a <_ZN2csC1Ei>

        test_func3();
      400cdc:    e8 47 ff ff ff         callq  400c28 <_Z10test_func3v>
      400ce1:    eb 17                  jmp    400cfa <_Z12test_func3_2v+0x56>
      400ce3:    48 89 45 b8            mov    %rax,0xffffffffffffffb8(%rbp)
      400ce7:    48 8b 5d b8            mov    0xffffffffffffffb8(%rbp),%rbx
      400ceb:    48 8d 7d c0            lea    0xffffffffffffffc0(%rbp),%rdi #c3的this指针
      400cef:    e8 2e 02 00 00         callq  400f22 <_ZN2csD1Ev>
      400cf4:    48 89 5d b8            mov    %rbx,0xffffffffffffffb8(%rbp)
      400cf8:    eb 0f                  jmp    400d09 <_Z12test_func3_2v+0x65>
      400cfa:    48 8d 7d c0            lea    0xffffffffffffffc0(%rbp),%rdi #c3的this指针
      400cfe:    e8 1f 02 00 00         callq  400f22 <_ZN2csD1Ev>
      400d03:    eb 17                  jmp    400d1c <_Z12test_func3_2v+0x78>
      400d05:    48 89 45 b8            mov    %rax,0xffffffffffffffb8(%rbp)
      400d09:    48 8b 5d b8            mov    0xffffffffffffffb8(%rbp),%rbx
      400d0d:    48 8d 7d d0            lea    0xffffffffffffffd0(%rbp),%rdi #c2的this指针
      400d11:    e8 0c 02 00 00         callq  400f22 <_ZN2csD1Ev>
      400d16:    48 89 5d b8            mov    %rbx,0xffffffffffffffb8(%rbp)
      400d1a:    eb 0f                  jmp    400d2b <_Z12test_func3_2v+0x87> 
      400d1c:    48 8d 7d d0            lea    0xffffffffffffffd0(%rbp),%rdi #c2的this指针
      400d20:    e8 fd 01 00 00         callq  400f22 <_ZN2csD1Ev>
      400d25:    eb 1e                  jmp    400d45 <_Z12test_func3_2v+0xa1>
      400d27:    48 89 45 b8            mov    %rax,0xffffffffffffffb8(%rbp)
      400d2b:    48 8b 5d b8            mov    0xffffffffffffffb8(%rbp),%rbx
      400d2f:    48 8d 7d e0            lea    0xffffffffffffffe0(%rbp),%rdi #c的this指针
      400d33:    e8 ea 01 00 00         callq  400f22 <_ZN2csD1Ev>
      400d38:    48 89 5d b8            mov    %rbx,0xffffffffffffffb8(%rbp)
      400d3c:    48 8b 7d b8            mov    0xffffffffffffffb8(%rbp),%rdi
      400d40:    e8 b3 fc ff ff         callq  4009f8 <_Unwind_Resume@plt>  #c的this指针
      400d45:    48 8d 7d e0            lea    0xffffffffffffffe0(%rbp),%rdi
      400d49:    e8 d4 01 00 00         callq  400f22 <_ZN2csD1Ev>
    }
      400d4e:    48 83 c4 48            add    $0x48,%rsp
      400d52:    5b                     pop    %rbx
      400d53:    c9                     leaveq 
      400d54:    c3                     retq   
      400d55:    90                     nop    


注意其中标红色的代码，\_ZN2csD1Ev即是类cs的析构函数，\_Unwind\_Resume()则是当清理完成时，用来从landing
pad返回的代码。test\_func3\_2()中只有3个cs
对象，但调用析构函数的代码却出现了6次。这里其实就是设置了多个出口函数，分别对应不同情况下，处理各个局部变量的析构，对于我们上面的代码来
说，test\_func3\_2()函数中的landing
pad就是从地址：400d09开始的，这些代码做了如下事情：

1. 先析构c2，然后jump到400d2b析构c.

2. 最后调用\_Unwind\_Resume（）

由此可见当程序中有多个可能抛异常的地方时，landing
pad也相应地会有多个，该函数的出口将更复杂，这也算是异常处理的一个overhead了。

总结
----

至此，关于GCC处理异常的具体流程及方式，各个细节都已写完，涉及很多比较琐碎的东西，只有反复阅读源码及相关文档才能搞明白，也不容易，只是古
人说的好，纸上得来终觉浅，为了加深印象及验证所学的内容，我根据前面了解的这些知识，简单仿着GCC写了一个简化版的c++
ABI，代码放到了github上[这里](https://github.com/kmalloc/SimpleCppExceptionHandling)，有兴趣的读者们可以参考一下，原本是打算把unwinder也写一遍的，但DWARF的格式实在太过复杂，已经超出了异常处理这个范围，就作罢了。

 

## 引用
- http://www.intel.com/content/dam/www/public/us/en/documents/guides/itanium-software-runtime-architecture-guide.pdf

- http://mentorembedded.github.io/cxx-abi/abi-eh.html

- http://refspecs.linuxfoundation.org/LSB\_3.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html

- https://www.opensource.apple.com/source/gcc/gcc-5341/gcc/

- http://www.cs.dartmouth.edu/\~sergey/battleaxe/hackito\_2011\_oakley\_bratus.pdf

- http://mentorembedded.github.io/cxx-abi/exceptions.pdf

- http://www.airs.com/blog/archives/464
