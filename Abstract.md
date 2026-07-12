## Title
Parameterized Machine Outlining for Similar Instruction Sequences

## Abstract
### Abstract for the Program Committee to properly consider your talk.

Machine outlining is a backend pass which identifies exact repeated code sequences and replaces them with calls to a shared function. Differential outlining generalizes this to similar, non-identical instruction sequences by normalizing equivalent patterns and parameterizing the differences that remain. Earlier prototype differential outliners relied on sequence alignment or similarity-preserving hashing approaches, which limited them to quadratic or cubic time costs. This talk introduces a novel MIR-level method that identifies and outlines similar instruction sequences in linear time.

The prototype was built around x86-64, though the boundary between the generic pass and the target-specific hooks was kept in mind at every step of the build, and so AArch64 has unimplemented versions of these target-specific functions.

Some of the features of the pass include (most of which are not possible with exact outlining):

- Outlining a majority of function calls: This pass has a separate analysis pass which strategically runs right before PEI, in order to relay information about whether each function callsite uses the stack to pass arguments. This is clearly important for outlining as a call to an outlined function changes the meaning of stack-accessing instructions in those outlined functions. This pass can outline most functions which do not pass arguments on the stack, while complying with stack offset rules.

- Parameterized immediates: This pass can parameterize most immediates, which are being used as either values or as displacements. I have built a process to find available registers in the intersection of all callsites in a group, then ensure that these registers are not overwritten from the callsite to the first usage of the parameter in the outlined region. There is a cool special case here, I call "parameter tunnelling". Let me show it to you:
```
Repeat 1:
  movq $5, %rdx
  more instructions ...
```
...
```
Repeat 2:
  movq $9, %rdx
  more instructions ...
```
Now if %rdx becomes the "carrier" register (carries the immediate from each callsite), then in the outlined function, the movq can be removed, since that instruction, in the outlined function, becomes: movq %rdx, %rdx. So in this case, the parameterization itself is free. From testing, about 25% of profitable outlined regions with parameters have a tunnelled parameter.
```
Repeat 1:
  movq $5, %rdx
  call outlined-function
```
...
```
Repeat 2:
  movq $9, %rdx
  call outlined-function
```
...
```
outlined-function:
  movq %rdx, %rdx <-- removed
  more instructions ...
```
- Stack-pointer-accessing instructions: Most instructions which read/modify the stack pointer are allowed to be outlined, with some restrictions. A couple of the main ones are: modifications must be known and net-zero across the outlined region, and functions using stack args are not allowed (though these are a minority). All stack pointer accesses have "fixups" depending on the semantics of the access, which allow them to be repaired in the outlined function. They can also have their immediates parameterized, as well as have their opcode canonicalized to an equivalent version, allowing them to be outlined/parameterized with fixups. A simple example of this in x86-64 is:
```
$r15 = MOV64rr $rsp
$r15 = LEA64r $rsp, 1, $noreg, [0 / can put a new offset here], $noreg
```

- The tokenization stage is built around making integer tokens, which are unique upto the abstract "shape" of an instruction. This abstraction allows for choosing what instructions are considered "similar". Two integer tokens are equal if their shapes are equal, which corresponds to their instructions being similar. Being able to decide this shape allows for fine-grained control of what we consider similar, and what components we know are repairable later on. It also gives an intuitive way to reduce further problems of finding similarities into: "Can you find a way to make the shapes match IFF you consider them similar?". The integer tokens are then fed to a suffix tree, much like the machine outliner, which efficiently finds exact repeated sequences. Since the abstraction is builtin to the token itself, exact repeats of tokens can represent similar sequences of instructions. This tokenization stage allows the pass to run with the same time complexity as the machine outliner.

- The main pass runs right before the machine outliner, which allows it to use the existing outlineable-region hooks, so targets don't need to re-implement them.

- There are more but this is getting long.

There are many ways by which to test a pass like this, but I will try to show a few:
Compiling the 25 largest files in llvm-project/build, -Oz:

```
 -118970  -9.810%  base= 1212754 param= 1093784  ELFDumper.cpp.o
 -116261 -11.367%  base= 1022760 param=  906499  AttrImpl.cpp.o
 -110700  -4.835%  base= 2289759 param= 2179059  Registry.cpp.o
  -82717  -4.811%  base= 1719509 param= 1636792  X86ISelLowering.cpp.o
  -65183  -3.714%  base= 1755202 param= 1690019  SemaOpenMP.cpp.o
  -58461  -3.332%  base= 1754757 param= 1696296  SemaExpr.cpp.o
  -55199  -7.009%  base=  787566 param=  732367  DynamicRecursiveASTVisitor.cpp.o
  -51298  -2.619%  base= 1958826 param= 1907528  SLPVectorizer.cpp.o
  -48318  -5.169%  base=  934821 param=  886503  AArch64ISelLowering.cpp.o
  -45372  -1.844%  base= 2461131 param= 2415759  AArch64MCTargetDesc.cpp.o
  -45361  -3.757%  base= 1207335 param= 1161974  SemaTemplateInstantiate.cpp.o
  -40137  -4.946%  base=  811501 param=  771364  ExprConstant.cpp.o
  -39978  -3.937%  base= 1015404 param=  975426  EvalEmitter.cpp.o
  -33870  -3.435%  base=  986167 param=  952297  DAGCombiner.cpp.o
  -26884  -2.770%  base=  970634 param=  943750  ASTReader.cpp.o
  -22655  -1.963%  base= 1154178 param= 1131523  PassBuilder.cpp.o
  -21671  -2.037%  base= 1063642 param= 1041971  Interp.cpp.o
  -20811  -2.657%  base=  783353 param=  762542  AArch64ISelDAGToDAG.cpp.o
  -20475  -1.471%  base= 1391628 param= 1371153  AArch64AsmParser.cpp.o
  -13955  -1.750%  base=  797375 param=  783420  AttributorAttributes.cpp.o
   -9128  -0.321%  base= 2845235 param= 2836107  X86MCTargetDesc.cpp.o
   -6001  -0.652%  base=  920277 param=  914276  Intrinsics.cpp.o
   -4661  -0.412%  base= 1132022 param= 1127361  X86AsmParser.cpp.o
   -1740  -0.060%  base= 2918890 param= 2917150  X86Disassembler.cpp.o
     -58  -0.006%  base= 1041272 param= 1041214  DiagnosticIDs.cpp.o
objects:     25
base text:   34935998
param text:  33876134
delta:       -1059864 bytes (-3.034%)
```

```
Top 60 / Bottom 10 from SingleSource/Benchmarks;MultiSource;MicroBenchmarks, -Oz
programs:    342
base text:   12203647
msp text:   11996924
delta:       -206723 bytes (-1.694%)
  -31127  -6.189%  base=  502979 msp=  471852  consumer-typeset
  -24242  -3.014%  base=  804240 msp=  779998  kc
  -12056  -1.969%  base=  612243 msp=  600187  clamscan
   -9897  -2.879%  base=  343712 msp=  333815  sqlite3
   -7636  -1.321%  base=  578251 msp=  570615  lencod
   -6328  -0.797%  base=  793899 msp=  787571  7zip-benchmark
   -6122  -1.287%  base=  475758 msp=  469636  CLAMR
   -4561  -3.077%  base=  148251 msp=  143690  lua
   -4264  -0.945%  base=  451091 msp=  446827  SPASS
   -3657  -1.283%  base=  285115 msp=  281458  pairlocalalign
   -3495  -1.639%  base=  213287 msp=  209792  timberwolfmc
   -3264  -1.929%  base=  169200 msp=  165936  gs
   -3191  -1.200%  base=  265894 msp=  262703  ldecod
   -2898  -0.464%  base=  624816 msp=  621918  bullet
   -2561  -1.485%  base=  172434 msp=  169873  smg2000
   -2240  -1.699%  base=  131878 msp=  129638  consumer-lame
   -2173  -2.844%  base=   76395 msp=   74222  paq8p
   -1760  -1.402%  base=  125496 msp=  123736  cjpeg
   -1644  -0.592%  base=  277614 msp=  275970  oggenc
   -1507  -1.314%  base=  114652 msp=  113145  treecc
   -1452  -1.071%  base=  135582 msp=  134130  espresso
   -1389  -1.149%  base=  120933 msp=  119544  consumer-jpeg
   -1240  -6.028%  base=   20572 msp=   19332  ControlFlow-flt
   -1204  -6.827%  base=   17635 msp=   16431  Reductions-flt
   -1196  -5.691%  base=   21015 msp=   19819  ControlFlow-dbl
   -1192  -2.690%  base=   44310 msp=   43118  miniGMG
   -1178  -2.043%  base=   57669 msp=   56491  office-ispell
   -1173  -3.016%  base=   38895 msp=   37722  alacconvert-decode
   -1173  -3.016%  base=   38895 msp=   37722  alacconvert-encode
   -1153  -7.024%  base=   16415 msp=   15262  Expansion-flt
   -1139  -2.829%  base=   40260 msp=   39121  simulator
   -1131  -6.857%  base=   16493 msp=   15362  ControlLoops-flt
   -1130  -6.283%  base=   17986 msp=   16856  Reductions-dbl
   -1130  -8.077%  base=   13990 msp=   12860  Equivalencing-flt
   -1116  -8.588%  base=   12995 msp=   11879  Packing-flt
   -1109  -7.046%  base=   15739 msp=   14630  GlobalDataFlow-flt
   -1109  -7.215%  base=   15370 msp=   14261  LoopRestructuring-flt
   -1108  -6.556%  base=   16900 msp=   15792  LinearDependence-flt
   -1106  -8.507%  base=   13001 msp=   11895  Recurrences-flt
   -1105  -6.583%  base=   16786 msp=   15681  Expansion-dbl
   -1103  -7.239%  base=   15236 msp=   14133  InductionVariable-flt
   -1102  -7.906%  base=   13939 msp=   12837  Symbolics-flt
   -1100  -8.464%  base=   12996 msp=   11896  StatementReordering-flt
   -1099  -7.770%  base=   14144 msp=   13045  NodeSplitting-flt
   -1094  -8.620%  base=   12691 msp=   11597  Searching-flt
   -1091  -8.038%  base=   13573 msp=   12482  LoopRerolling-flt
   -1088  -6.448%  base=   16873 msp=   15785  ControlLoops-dbl
   -1087  -7.557%  base=   14384 msp=   13297  IndirectAddressing-flt
   -1078  -6.854%  base=   15729 msp=   14651  LoopRestructuring-dbl
   -1066  -7.426%  base=   14355 msp=   13289  Equivalencing-dbl
   -1065  -6.157%  base=   17296 msp=   16231  LinearDependence-dbl
   -1061  -6.592%  base=   16096 msp=   15035  GlobalDataFlow-dbl
   -1060  -6.795%  base=   15600 msp=   14540  InductionVariable-dbl
   -1060  -7.304%  base=   14512 msp=   13452  NodeSplitting-dbl
   -1059  -7.105%  base=   14906 msp=   13847  CrossingThresholds-flt
   -1059  -7.930%  base=   13355 msp=   12296  Packing-dbl
   -1056  -7.909%  base=   13352 msp=   12296  Recurrences-dbl
   -1056  -8.092%  base=   13050 msp=   11994  Searching-dbl
   -1055  -7.381%  base=   14293 msp=   13238  Symbolics-dbl
   -1050  -7.867%  base=   13347 msp=   12297  StatementReordering-dbl
     +23  +0.436%  base=    5272 msp=    5295  fixoutput
     +24  +0.474%  base=    5064 msp=    5088  automotive-bitcount
     +25  +0.104%  base=   24057 msp=   24082  stepanov_container
     +25  +0.454%  base=    5507 msp=    5532  trees
     +40  +0.277%  base=   14451 msp=   14491  stepanov_abstraction
     +46  +0.503%  base=    9145 msp=    9191  deriv2
     +47  +0.634%  base=    7414 msp=    7461  ocean
     +64  +0.554%  base=   11542 msp=   11606  bigfib
    +147  +0.756%  base=   19447 msp=   19594  stepanov_vector
    +658  +0.315%  base=  208817 msp=  209475  spirit
```


## Website/Youtube Abstract
### 1 paragraph. This is displayed on the schedule and website for attendees to consider when selecting talks. It is also used for the YouTube posting of your video. We suggest you proofread and pay attention to grammar.
Machine outlining is a backend pass which identifies exact repeated code sequences and replaces them with calls to a shared function. Differential outlining generalizes this to similar, non-identical instruction sequences by normalizing equivalent patterns and parameterizing the differences that remain. Earlier prototype differential outliners relied on sequence alignment or similarity-preserving hashing approaches, which limited them to quadratic or cubic time costs. This talk introduces a novel MIR-level method that identifies and outlines similar instruction sequences in linear time.


## Speaker 1 - Bio (hidden from reviewers)
### 1 paragraph bio for Speaker (Viewable by admins only). This is used for speaker registration and for our speaker page on the event site.
Undergraduate mathematical physics student at the University of Waterloo. Can be nerd sniped into anything, including building differential outliners.
