# lua-prevent-of-decompilation
lua的一个简单的预防反编译的方法。

此方法主要做了两次防御：
* 变更操作码指令顺序。
* 在输入输出字节码的时候进行异或处理。

## 变更操作码指令顺序
提供了一个[excel文件（lua_opcode乱序.xlsx）](lua_opcode乱序.xlsx)自动把指令顺序打乱，需要修改的文件是"lopcodes.h"、"lopcodes.c"文件。变更类型: "luaP_opnames"、"luaP_opmodes"、"OpCode"，具体请看code目录下的两个文件。

## 在输入输出字节码的时候进行循环异或处理
如果代码是明文，lua在执行时会直接解释成操作码。项目中一般为了避免明文代码泄露会用luacompiler先解释成字节码文件，lua能区分明文和密文的主要区别就是文件头的12个字节，具体可以看``static void DumpHeader(DumpState* D)``和``static void LoadHeader(LoadState* S)``函数,所以这12个字节没有对其加密。

仔细观察，lua其实是把每个文件都看待成一个funcition（lua中function其实和闭包是一样的）。
```
static void DumpFunction(const Proto* f, const TString* p, DumpState* D)
{
 DumpString((f->source==p || D->strip) ? NULL : f->source,D);
 DumpInt(f->linedefined,D);
 DumpInt(f->lastlinedefined,D);
 DumpChar(f->nups,D);
 DumpChar(f->numparams,D);
 DumpChar(f->is_vararg,D);
 DumpChar(f->maxstacksize,D);
 DumpCode(f,D);
 DumpConstants(f,D);
 DumpDebug(f,D);
}
```
观察``DumpFunction``的实现，闭包中的源码字符串、开始行号、结束行号、upvalue数量（闭包参数）、函数参数数量、栈大小、字节码、常量列表、debug信息。（其中is_vararg没看明白是什么，赋值是VARARG_HASARG、VARARG_ISVARARG、VARARG_NEEDSARG或运算求得的，估计和函数参数有关系。）

在lua里面，大概就几种基础的数据类型，"ldump.c"下就如下几种数据类型的实现：char、int、number、vector、string，这几种数据类型最终还是会通过``static void DumpBlock(const void* b, size_t size, DumpState* D)``函数写入到文件，通过``static void LoadBlock(LoadState* S, void* b, size_t size)``函数加载到内存。下面是这个两个函数改动，writer前进去加密，在read后进行解密。

```
static void DumpBlock(const void* b, size_t size, DumpState* D)
{
 if (D->status==0)
 {
  XorBlockEncrypt(b,size);
  lua_unlock(D->L);
  D->status=(*D->writer)(D->L,b,size,D->data);
  lua_lock(D->L);
  XorBlockDecrypt(b,size);
 }
}

static void LoadBlock(LoadState* S, void* b, size_t size)
{
 size_t r=luaZ_read(S->Z,b,size);
 XorBlockDecrypt(b,size);
 IF (r!=0, "unexpected end");
}

#define XorBlockEncrypt(b,s)		{size_t i;for(i=1;i<s;++i){cast(char*,b)[i]=cast(char*,b)[i]^cast(char*,b)[i-1];}}
#define XorBlockDecrypt(b,s)		{size_t i;for(i=(s>0?s-1:0);i>0;--i){cast(char*,b)[i]=cast(char*,b)[i]^cast(char*,b)[i-1];}}

``

