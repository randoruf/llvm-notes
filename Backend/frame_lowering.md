# FrameLowering

[LLVM: lib/Target/X86/X86FrameLowering.cpp Source File](https://llvm.org/doxygen/X86FrameLowering_8cpp_source.html#l02495)

```cpp
   // LLVM arranges the stack as follows:
   //   ...
   //   ARG2
   //   ARG1
   //   RETADDR
   //   PUSH RBP   <-- RBP points here
   //   PUSH CSRs
   //   ~~~~~~~    <-- possible stack realignment (non-win64)
   //   ...
   //   STACK OBJECTS
   //   ...        <-- RSP after prologue points here
   //   ~~~~~~~    <-- possible stack realignment (win64)
   //
```

```cpp
   // if (hasVarSizedObjects()):
   //   ...        <-- "base pointer" (ESI/RBX) points here
   //   DYNAMIC ALLOCAS
   //   ...        <-- RSP points here
   //
   // Case 1: In the simple case of no stack realignment and no dynamic
   // allocas, both "fixed" stack objects (arguments and CSRs) are addressable
   // with fixed offsets from RSP.
   //
   // Case 2: In the case of stack realignment with no dynamic allocas, fixed
   // stack objects are addressed with RBP and regular stack objects with RSP.
   //
   // Case 3: In the case of dynamic allocas and stack realignment, RSP is used
   // to address stack arguments for outgoing calls and nothing else. The "base
   // pointer" points to local variables, and RBP points to fixed objects.
   //
   // In cases 2 and 3, we can only answer for non-fixed stack objects, and the
   // answer we give is relative to the SP after the prologue, and not the
   // SP in the middle of the function.
```

### Code Emit

[LLVM: lib/Target/X86/X86FrameLowering.cpp Source File](https://llvm.org/doxygen/X86FrameLowering_8cpp_source.html#l01394)

```cpp
/*
   Here's a gist of what gets emitted:
  
   ; Establish frame pointer, if needed
   [if needs FP]
       push  %rbp
       .cfi_def_cfa_offset 16
       .cfi_offset %rbp, -16
       .seh_pushreg %rpb
       mov  %rsp, %rbp
       .cfi_def_cfa_register %rbp
  
   ; Spill general-purpose registers
   [for all callee-saved GPRs]
       pushq %<reg>
       [if not needs FP]
          .cfi_def_cfa_offset (offset from RETADDR)
       .seh_pushreg %<reg>
  
   ; If the required stack alignment > default stack alignment
   ; rsp needs to be re-aligned.  This creates a "re-alignment gap"
   ; of unknown size in the stack frame.
   [if stack needs re-alignment]
       and  $MASK, %rsp
  
   ; Allocate space for locals
   [if target is Windows and allocated space > 4096 bytes]
       ; Windows needs special care for allocations larger
       ; than one page.
       mov $NNN, %rax
       call ___chkstk_ms/___chkstk
       sub  %rax, %rsp
   [else]
       sub  $NNN, %rsp
  
   [if needs FP]
       .seh_stackalloc (size of XMM spill slots)
       .seh_setframe %rbp, SEHFrameOffset ; = size of all spill slots
   [else]
       .seh_stackalloc NNN
  
   ; Spill XMMs
   ; Note, that while only Windows 64 ABI specifies XMMs as callee-preserved,
   ; they may get spilled on any platform, if the current function
   ; calls @llvm.eh.unwind.init
   [if needs FP]
       [for all callee-saved XMM registers]
           movaps  %<xmm reg>, -MMM(%rbp)
       [for all callee-saved XMM registers]
           .seh_savexmm %<xmm reg>, (-MMM + SEHFrameOffset)
               ; i.e. the offset relative to (%rbp - SEHFrameOffset)
   [else]
       [for all callee-saved XMM registers]
           movaps  %<xmm reg>, KKK(%rsp)
       [for all callee-saved XMM registers]
           .seh_savexmm %<xmm reg>, KKK
  
   .seh_endprologue
  
   [if needs base pointer]
       mov  %rsp, %rbx
       [if needs to restore base pointer]
           mov %rsp, -MMM(%rbp)
  
   ; Emit CFI info
   [if needs FP]
       [for all callee-saved registers]
           .cfi_offset %<reg>, (offset from %rbp)
   [else]
        .cfi_def_cfa_offset (offset from RETADDR)
       [for all callee-saved registers]
           .cfi_offset %<reg>, (offset from %rsp)
  
   Notes:
   - .seh directives are emitted only for Windows 64 ABI
   - .cv_fpo directives are emitted on win32 when emitting CodeView
   - .cfi directives are emitted for all other ABIs
   - for 32-bit code, substitute %e?? registers for %r??
 */
```

