## SYCL  Clang 源码分析

Clang详尽流程图

首先，C代码程序文件，然后经过了Clang前端（或者古老的llvm-gcc前端，这是在Clang出现以前使用的前端，可以产生LLVM IR），然后会产生LLVM IR。

LLVM IR是以.bc结尾，缩写代表的单词是bitcode，它是序列化的数据，用以存储在磁盘中。

三种LLVM IR表示格式，除了bitcode这种存储格式，第二种是可读格式，一般以.ll结尾，第三种格式则是内存表示格式，这一般是用于我们程序开发，去操作LLVM IR。

产生.bc的阶段。产生完毕后，它这里直接进入llvm-link阶段，其实不太完整。若是我们使用了O2等优化，我们在这里还会经历一个优化的阶段，即对这一个编译单元的优化，大约包括了40多种优化Pass，如死代码消除，内联等。而经过优化后，依然是.bc文件，然后再进入llvm-link阶段。

而在llvm-link阶段，所要做的其实就是把多个.bc文件合并成一个.bc文件，并且做链接时优化。

经过llvm-link阶段后，我们得到了一个新的.bc，那么我们会再次进入优化阶段，重新再进行一次优化。因为经过llvm-link后，我们的IR结构可能发生了一些变化，从而可能更有利于我们优化，得到更优化的IR。而这一步骤后，可以得到最终优化后的.bc，正式进入代码生成阶段。

这里只列举出来了一个llc，即走机器代码生成的路径。其实对于LLVM来说，还可以有一个过程，那就是lli，即解释执行LLVM IR。

llc的作用就是把LLVM IR 编译成汇编文件.s，产生了汇编文件以后，就可以调用系统的汇编器，如GNU的as，从而产生目标文件object file，即.o文件了。

另一条路llvm-mc，是直接走生成object file的过程。而这条路径在如今来说，并不是每个平台都会走，核心在于LLVM本身的集成汇编器，即-fintegrated-as，使用集成汇编器既可以使用MCLoweing到MCInst，再使用MCStreamers直接到.o，当然也可以到.s。

而生成完.o文件，链接器链接相关的库，然后和目标文件一起生存可执行文件.out / .exe了

![Clang详尽流程图](https://pic3.zhimg.com/80/6a1eb1578a398a1c7788bc91618647ea_1440w.webp)



驱动前端大体流程：Parse/Pipeline/Bind/Translate/Execute等过程，从外部Driver层真正的走入到Clang核心前端层（即cc1）中，从而执行编译等过程。

![clang Driver设计图](https://pic4.zhimg.com/80/v2-ec299786d7d45a2dc0e25dc32315c473_1440w.webp)

这张图的橘色部分除了代表图片左上角提及的Input / Output，更是代表了Clang Driver里面的具体数据结构，如你所见的ArgList；绿色部分除了代表 Driver Functions，也代表了Driver的具体走向流程；蓝紫色部分则是一些重要的辅助类。



Parse的作用是选项解析，负责把用户传入过来的命令行选项解析成一个一个的参数，放入到Arg实例中。Pipeline配合HostInfo吃的ArgList & Args，吐出的是Actions，然后Bind ／ Translate是配合了ToolChain / Tools 吃掉Actions，吐出Jobs。Execute则是拿到Jobs，返回Result Code。

1. 入口函数 main

   clang/tools/driver/driver.cpp

   ```cpp
   int main(int argc_, const char **argv_) {
       ...
   // Handle -cc1 integrated tools, even if -cc1 was expanded from a response
     // file.
     auto FirstArg = std::find_if(argv.begin() + 1, argv.end(),
                                  [](const char *A) { return A != nullptr; });
     if (FirstArg != argv.end() && StringRef(*FirstArg).startswith("-cc1")) {
       // If -cc1 came from a response file, remove the EOL sentinels.
       if (MarkEOLs) {
         auto newEnd = std::remove(argv.begin(), argv.end(), nullptr);
         argv.resize(newEnd - argv.begin());
       }
       //在经历过execve创建子进程调用执行时或使用-cc1参数直接进入Clang前端，会根据-cc1进入到这里执行
       // -cc1 调用栈逻辑 1->2-3->4->5
       return ExecuteCC1Tool(argv, argv[1] + 4);
     }
       ...
     Driver TheDriver(Path, llvm::sys::getDefaultTargetTriple(), Diags);
     SetInstallDir(argv, TheDriver, CanonicalPrefixes);
     TheDriver.setTargetAndMode(TargetAndMode);
   
     insertTargetAndModeArgs(TargetAndMode, argv, SavedStrings);
   
     SetBackdoorDriverOutputsFromEnvVars(TheDriver);
     //build compilation 
     // Compilation - A set of tasks to perform for a single driver invocation.
     // BuildCompilation 进入 8
     // 调用栈的顺序 8->9/10->11->14->17/18 
     // 调用栈的回溯 17/18 ->14->11->9/10->8
     std::unique_ptr<Compilation> C(TheDriver.BuildCompilation(argv)); 
     int Res = 1;
     bool IsCrash = false;
     if (C && !C->containsError()) {
       SmallVector<std::pair<int, const Command *>, 4> FailingCommands;
       //build后拿到Job，然后去执行编译的步骤 进入 19
       //这里添加了-cc1的执行调用栈顺序 是 19->20->21->22
       //然后启动多进程
       Res = TheDriver.ExecuteCompilation(*C, FailingCommands);
       ...
     }
   }
   ```

   在这里初始化了Driver，并且调用BuildCompilation函数，然后生成一个Compilation.

   clang/include/clang/Driver/Compilation.h

   ```cpp
   /// Compilation - A set of tasks to perform for a single driver
   /// invocation.
   class Compilation {
     /// The driver we were created by.
     const Driver &TheDriver;
   
     /// The default tool chain.
     const ToolChain &DefaultToolChain;
   
     /// A mask of all the programming models the host has to support in the
     /// current compilation.
     unsigned ActiveOffloadMask;
   
     /// Array with the toolchains of offloading host and devices in the order they
     /// were requested by the user. We are preserving that order in case the code
     /// generation needs to derive a programming-model-specific semantic out of
     /// it.
     std::multimap<Action::OffloadKind, const ToolChain *>
         OrderedOffloadingToolchains;
   
     /// The original (untranslated) input argument list.
     llvm::opt::InputArgList *Args;
   
     /// The driver translated arguments. Note that toolchains may perform their
     /// own argument translation.
     llvm::opt::DerivedArgList *TranslatedArgs;
   
     /// The list of actions we've created via MakeAction.  This is not accessible
     /// to consumers; it's here just to manage ownership.
     std::vector<std::unique_ptr<Action>> AllActions;
   
     /// The list of actions.  This is maintained and modified by consumers, via
     /// getActions().
     ActionList Actions;
   
     /// The root list of jobs.
     JobList Jobs;
   
     /// Cache of translated arguments for a particular tool chain and bound
     /// architecture.
     llvm::DenseMap<std::pair<const ToolChain *, const char *>,
                    llvm::opt::DerivedArgList *> TCArgs;
   
     /// Temporary files which should be removed on exit.
     llvm::opt::ArgStringList TempFiles;
   
     /// Result files which should be removed on failure.
     ArgStringMap ResultFiles;
   
     /// Result files which are generated correctly on failure, and which should
     /// only be removed if we crash.
     ArgStringMap FailureResultFiles;
   
     /// Redirection for stdout, stderr, etc.
     const StringRef **Redirects;
   
     /// Whether we're compiling for diagnostic purposes.
     bool ForDiagnostics;
       ...
   }
   ```

   clang/tools/driver/driver.cpp

   int ExecuteCC1Tool(ArrayRef<const char *> argv, StringRef Tool)

   ```cpp
   static int ExecuteCC1Tool(ArrayRef<const char *> argv, StringRef Tool) {
     void *GetExecutablePathVP = (void *)(intptr_t) GetExecutablePath;
     if (Tool == "")
       return cc1_main(argv.slice(2), argv[0], GetExecutablePathVP);
     if (Tool == "as")
       return cc1as_main(argv.slice(2), argv[0], GetExecutablePathVP);
     if (Tool == "gen-reproducer")
       return cc1gen_reproducer_main(argv.slice(2), argv[0], GetExecutablePathVP);
   
     // Reject unknown tools.
     llvm::errs() << "error: unknown integrated tool '" << Tool << "'. "
                  << "Valid tools include '-cc1' and '-cc1as'.\n";
     return 1;
   }
   ```

   

2. 进入cc1_main()函数

   clang/tools/driver/cc1_main.cpp

   ```cpp
   int cc1_main(ArrayRef<const char *> Argv, const char *Argv0, void *MainAddr){
     ensureSufficientStack();
   // CompilerInstance - Helper class for managing a single instance of the Clang compiler.
     //clang/CompilerInstance.h
     std::unique_ptr<CompilerInstance> Clang(new CompilerInstance());
     IntrusiveRefCntPtr<DiagnosticIDs> DiagID(new DiagnosticIDs());
     //默认clang cc1调用的invocation 转 6  CompilerInvocation::CreateFromArgs()
     bool Success =
         CompilerInvocation::CreateFromArgs(Clang->getInvocation(), Argv, Diags);
     ...
   // Execute the frontend actions.
     {
       llvm::TimeTraceScope TimeScope("ExecuteCompiler", StringRef(""));
       Success = ExecuteCompilerInvocation(Clang.get());
     }      
   }
   ```

   

3. 进入ExecuteCompilerInvocation(Clang.get())函数

   clang/ExecuteCompilerInvocation.cpp

   ```cpp
   bool ExecuteCompilerInvocation(CompilerInstance *Clang) {
     ...
     // Create and execute the frontend action.
     std::unique_ptr<FrontendAction> Act(CreateFrontendAction(*Clang));
     if (!Act)
       return false;
     bool Success = Clang->ExecuteAction(*Act);
     ...
   }
   ```

   

4. 进入CreateFrontAction(*Clang)函数

   clang/ExecuteCompilerInvocation.cpp

   ```cpp
   std::unique_ptr<FrontendAction> CreateFrontendAction(CompilerInstance &CI) {
     // Create the underlying action.
     std::unique_ptr<FrontendAction> Act = CreateFrontendBaseAction(CI);
     if (!Act)
       return nullptr; 
   }
   ```

   

5. 进入CreateFrontendBaseAction函数

   clang/lib/FrontTool/ExecuteCompilerInvocation.cpp

   ```cpp
   static std::unique_ptr<FrontendAction> CreateFrontendBaseAction(CompilerInstance &CI) {
     using namespace clang::frontend;
     StringRef Action("unknown");
     (void)Action;
   
     switch (CI.getFrontendOpts().ProgramAction) {
     case ASTDeclList:            return std::make_unique<ASTDeclListAction>();
     case ASTDump:                return std::make_unique<ASTDumpAction>();
     case ASTPrint:               return std::make_unique<ASTPrintAction>();
     case ASTView:                return std::make_unique<ASTViewAction>();
     case DumpCompilerOptions:
       return std::make_unique<DumpCompilerOptionsAction>();
     case DumpRawTokens:          return std::make_unique<DumpRawTokensAction>();
     case DumpTokens:             return std::make_unique<DumpTokensAction>();
     case EmitAssembly:           return std::make_unique<EmitAssemblyAction>();
     case EmitBC:                 return std::make_unique<EmitBCAction>();
     case EmitHTML:               return std::make_unique<HTMLPrintAction>();
     case EmitLLVM:               return std::make_unique<EmitLLVMAction>();
     case EmitLLVMOnly:           return std::make_unique<EmitLLVMOnlyAction>();
     case EmitCodeGenOnly:        return std::make_unique<EmitCodeGenOnlyAction>();
     case EmitObj:                return std::make_unique<EmitObjAction>();
     case FixIt:                  return std::make_unique<FixItAction>();
     case GenerateModule:
       return std::make_unique<GenerateModuleFromModuleMapAction>();
     case GenerateModuleInterface:
       return std::make_unique<GenerateModuleInterfaceAction>();
     case GenerateHeaderModule:
       return std::make_unique<GenerateHeaderModuleAction>();
     case GeneratePCH:            return std::make_unique<GeneratePCHAction>();
     case GenerateInterfaceIfsExpV1:
       return std::make_unique<GenerateInterfaceIfsExpV1Action>();
     case InitOnly:               return std::make_unique<InitOnlyAction>();
     //Clang cc1 默认的ProgramAction          
     case ParseSyntaxOnly:        return std::make_unique<SyntaxOnlyAction>();
     case ModuleFileInfo:         return std::make_unique<DumpModuleInfoAction>();
     case VerifyPCH:              return std::make_unique<VerifyPCHAction>();
     case TemplightDump:          return std::make_unique<TemplightDumpAction>();     ...
   }
   ```

   

6. 进入CompilerInvocation::CreateFromArgs()函数

   clang/lib/Frontend/CompilerInvocation.cpp

   ```cpp
   bool CompilerInvocation::CreateFromArgs(CompilerInvocation &Res,
                                           ArrayRef<const char *> CommandLineArgs,
                                           DiagnosticsEngine &Diags) {
   // FIXME: We shouldn't have to pass the DashX option around here
     InputKind DashX = ParseFrontendArgs(Res.getFrontendOpts(), Args, Diags,
                                         LangOpts.IsHeaderFile);    
   }
   ```

   

7. 进入ParseFrontendArgs()函数

   clang/lib/Frontend/CompilerInvocation.cpp

   ```cpp
   static InputKind ParseFrontendArgs(FrontendOptions &Opts, ArgList &Args,
                                      DiagnosticsEngine &Diags) {
     //默认ProgramAction是 ParseSyntaxOnly    
     Opts.ProgramAction = frontend::ParseSyntaxOnly;
     if (const Arg *A = Args.getLastArg(OPT_Action_Group)) {
       switch (A->getOption().getID()) {
       default:
         llvm_unreachable("Invalid option in group!");
       case OPT_ast_list:
         Opts.ProgramAction = frontend::ASTDeclList; break;
       case OPT_ast_dump_all_EQ:
       case OPT_ast_dump_EQ: {
         unsigned Val = llvm::StringSwitch<unsigned>(A->getValue())
                            .CaseLower("default", ADOF_Default)
                            .CaseLower("json", ADOF_JSON)
                            .Default(std::numeric_limits<unsigned>::max());
   
         if (Val != std::numeric_limits<unsigned>::max())
           Opts.ASTDumpFormat = static_cast<ASTDumpOutputFormat>(Val);
         else {
           Diags.Report(diag::err_drv_invalid_value)
               << A->getAsString(Args) << A->getValue();
           Opts.ASTDumpFormat = ADOF_Default;
         }
         LLVM_FALLTHROUGH;
       }
       case OPT_ast_dump:
       case OPT_ast_dump_all:
       case OPT_ast_dump_lookups:
         Opts.ProgramAction = frontend::ASTDump; break;
       case OPT_ast_print:
         Opts.ProgramAction = frontend::ASTPrint; break;
       case OPT_ast_view:
         Opts.ProgramAction = frontend::ASTView; break;
       case OPT_compiler_options_dump:
         Opts.ProgramAction = frontend::DumpCompilerOptions; break;
       case OPT_dump_raw_tokens:
         Opts.ProgramAction = frontend::DumpRawTokens; break;
       case OPT_dump_tokens:
         Opts.ProgramAction = frontend::DumpTokens; break;
       case OPT_S:
         Opts.ProgramAction = frontend::EmitAssembly; break;
       case OPT_emit_llvm_bc:
         Opts.ProgramAction = frontend::EmitBC; break;
       case OPT_emit_html:
         Opts.ProgramAction = frontend::EmitHTML; break;
       case OPT_emit_llvm:
         Opts.ProgramAction = frontend::EmitLLVM; break;
       case OPT_emit_llvm_only:
         Opts.ProgramAction = frontend::EmitLLVMOnly; break;
       case OPT_emit_codegen_only:
         Opts.ProgramAction = frontend::EmitCodeGenOnly; break;
       case OPT_emit_obj:
         Opts.ProgramAction = frontend::EmitObj; break;
   ...            
   }
   ```

   

8. 构建编译Driver::BuildCompilation()

   clang/lib/Driver/Driver.cpp

   ```cpp
   Compilation *Driver::BuildCompilation(ArrayRef<const char *> ArgList) {
     // Arguments specified in command line.
     bool ContainsError;
     /// Arguments originated from command line. clang/include/clang/Driver/Driver.h
     /// std::unique_ptr<llvm::opt::InputArgList> CLOptions; 
     /// Parse 操作 主要部分 通过Input File 以及ArgList命令行参数输入，输出InputArgList以及Args
     CLOptions = std::make_unique<InputArgList>(
         ParseArgStrings(ArgList.slice(1), IsCLMode(), ContainsError));
     // Try parsing configuration file.
     if (!ContainsError)
       ContainsError = loadConfigFile();
     bool HasConfigFile = !ContainsError && (CfgOptions.get() != nullptr);
   
     // All arguments, from both config file and command line.
     InputArgList Args = std::move(HasConfigFile ? std::move(*CfgOptions)
                                                 : std::move(*CLOptions));  
     ...    
     bool CCCPrintPhases;
   
     // Silence driver warnings if requested
     Diags.setIgnoreAllWarnings(Args.hasArg(options::OPT_w));
   
     // -no-canonical-prefixes is used very early in main.
     Args.ClaimAllArgs(options::OPT_no_canonical_prefixes);
   
     // f(no-)integated-cc1 is also used very early in main.
     Args.ClaimAllArgs(options::OPT_fintegrated_cc1);
     Args.ClaimAllArgs(options::OPT_fno_integrated_cc1);
   
     // Ignore -pipe.
     Args.ClaimAllArgs(options::OPT_pipe);
   
     // Extract -ccc args.
     //
     // FIXME: We need to figure out where this behavior should live. Most of it
     // should be outside in the client; the parts that aren't should have proper
     // options, either by introducing new ones or by overloading gcc ones like -V
     // or -b.
     CCCPrintPhases = Args.hasArg(options::OPT_ccc_print_phases);
     CCCPrintBindings = Args.hasArg(options::OPT_ccc_print_bindings);
     if (const Arg *A = Args.getLastArg(options::OPT_ccc_gcc_name))
       CCCGenericGCCName = A->getValue(); 
     //Parse 操作 另一部分  
     for (const Arg *A : Args.filtered(options::OPT_B)) {
       A->claim();
       PrefixDirs.push_back(A->getValue(0));
     }
     if (Optional<std::string> CompilerPathValue =
             llvm::sys::Process::GetEnv("COMPILER_PATH")) {
       StringRef CompilerPath = *CompilerPathValue;
       while (!CompilerPath.empty()) {
         std::pair<StringRef, StringRef> Split =
             CompilerPath.split(llvm::sys::EnvPathSeparator);
         PrefixDirs.push_back(std::string(Split.first));
         CompilerPath = Split.second;
       }
     }
     if (const Arg *A = Args.getLastArg(options::OPT__sysroot_EQ))
       SysRoot = A->getValue();
     if (const Arg *A = Args.getLastArg(options::OPT__dyld_prefix_EQ))
       DyldPrefix = A->getValue();
   
     if (const Arg *A = Args.getLastArg(options::OPT_resource_dir))
       ResourceDir = A->getValue();
   
     if (const Arg *A = Args.getLastArg(options::OPT_save_temps_EQ)) {
       SaveTemps = llvm::StringSwitch<SaveTempsMode>(A->getValue())
                       .Case("cwd", SaveTempsCwd)
                       .Case("obj", SaveTempsObj)
                       .Default(SaveTempsCwd);
     }    
   }
   ```

   

9. Pipeline配合HostInfo拿到相关的ArgsList & Args，根据不同的编译选项，进行不同的Compiler Action

   首先我们会拿到不同平台的ToolChain，如Linux / Darwin / Win32等，然后放入到Compilation对象中，可以看到我们注重的BuildUniversalActions（Mac OS X）／ BuildActions (Others Platform)，即Pipeline的结果，并且这里表明了我们如果有CCCPrintPhases，就会执行PrintActions(*C)。而在BuildActions之前还有一个BuildInputs，则对应于我们说过的特殊Action：InputAction。在那里则会检测编译选项（如-x）和文件后缀名等，从而让编译器知道目前编译的是什么源文件，如是C还是C++

   clang/lib/Driver/Driver.cpp

   

   ```cpp
   Compilation *Driver::BuildCompilation(ArrayRef<const char *> ArgList) {
     /// Parse 操作 主要部分 通过Input File 以及ArgList命令行参数输入，输出InputArgList以及Args
     CLOptions = std::make_unique<InputArgList>(
         ParseArgStrings(ArgList.slice(1), IsCLMode(), ContainsError)); 
     ...
     //先拿到ToolChain      
     // Owned by the host.
     const ToolChain &TC = getToolChain(
         *UArgs, computeTargetTriple(*this, TargetTriple, *UArgs));
     //放到Compilation中
     // The compilation takes ownership of Args.
     Compilation *C = new Compilation(*this, TC, UArgs.release(), TranslatedArgs,
                                      ContainsError);
   
     if (!HandleImmediateArgs(*C))
       return C;
   
     // Construct the list of inputs.
     //构建特殊的Action InputAction
     InputList Inputs;
     BuildInputs(C->getDefaultToolChain(), *TranslatedArgs, Inputs);
   
     // Determine if there are any offload static libraries.
     if (checkForOffloadStaticLib(*C, *TranslatedArgs))
       setOffloadStaticLibSeen();
   
     // Populate the tool chains for the offloading devices, if any.
     CreateOffloadingDeviceToolChains(*C, Inputs);    
     ...
     // Construct the list of abstract actions to perform for this compilation. On
     // MachO targets this uses the driver-driver and universal actions.
     if (TC.getTriple().isOSBinFormatMachO())
       BuildUniversalActions(*C, C->getDefaultToolChain(), Inputs);
     else
       BuildActions(*C, C->getArgs(), Inputs, C->getActions());  
     //如果打印printPhases 则打印  
     if (CCCPrintPhases) {
       PrintActions(*C);
       return C;
     }
   
     BuildJobs(*C);      
   }
   ```

   

10. 进入BuildActions 

    clang/lib/Driver/Driver.cpp  OffloadBuilder->processHostLinkAction(LA);

    ```cpp
    void Driver::BuildActions(Compilation &C, DerivedArgList &Args,
                              const InputList &Inputs, ActionList &Actions) const {
      llvm::PrettyStackTraceString CrashInfo("Building compilation actions");
    
      handleArguments(C, Args, Inputs, Actions);
      ...
      // Add an interface stubs merge action if necessary.
      if (!MergerInputs.empty())
        Actions.push_back(
            C.MakeAction<IfsMergeJobAction>(MergerInputs, types::TY_Image));    
    }
    
    void Driver::handleArguments(Compilation &C, DerivedArgList &Args,
                                 const InputList &Inputs,
                                 ActionList &Actions) const {
      ...    
      Arg *FinalPhaseArg;
      phases::ID FinalPhase = getFinalPhase(Args, &FinalPhaseArg);
    
      if (FinalPhase == phases::Link) {
        if (Args.hasArg(options::OPT_emit_llvm))
          Diag(clang::diag::err_drv_emit_llvm_link);
        if (IsCLMode() && LTOMode != LTOK_None &&
            !Args.getLastArgValue(options::OPT_fuse_ld_EQ).equals_lower("lld"))
          Diag(clang::diag::err_drv_lto_without_lld);
      }
        
      if (FinalPhase == phases::Link) {
        if (Args.hasArg(options::OPT_emit_llvm))
          Diag(clang::diag::err_drv_emit_llvm_link);
        if (IsCLMode() && LTOMode != LTOK_None &&
            !Args.getLastArgValue(options::OPT_fuse_ld_EQ).equals_lower("lld"))
          Diag(clang::diag::err_drv_lto_without_lld);
      }
    
      if (FinalPhase == phases::Preprocess || Args.hasArg(options::OPT__SLASH_Y_)) {
        // If only preprocessing or /Y- is used, all pch handling is disabled.
        // Rather than check for it everywhere, just remove clang-cl pch-related
        // flags here.
        Args.eraseArg(options::OPT__SLASH_Fp);
        Args.eraseArg(options::OPT__SLASH_Yc);
        Args.eraseArg(options::OPT__SLASH_Yu);
        YcArg = YuArg = nullptr;
      }   
      ...    
    }
    
    // 决定最后走到哪个Phase
    // Determine which compilation mode we are in. We look for options which
    // affect the phase, starting with the earliest phases, and record which
    // option we used to determine the final phase.
    phases::ID Driver::getFinalPhase(const DerivedArgList &DAL,
                                     Arg **FinalPhaseArg) const {
      Arg *PhaseArg = nullptr;
      phases::ID FinalPhase;
      //预处理
      // -{E,EP,P,M,MM} only run the preprocessor.
      if (CCCIsCPP() || (PhaseArg = DAL.getLastArg(options::OPT_E)) ||
          (PhaseArg = DAL.getLastArg(options::OPT__SLASH_EP)) ||
          (PhaseArg = DAL.getLastArg(options::OPT_M, options::OPT_MM)) ||
          (PhaseArg = DAL.getLastArg(options::OPT__SLASH_P))) {
        FinalPhase = phases::Preprocess;
      //预编译
      // --precompile only runs up to precompilation.
      } else if ((PhaseArg = DAL.getLastArg(options::OPT__precompile))) {
        FinalPhase = phases::Precompile;
      //编译 - 语法分析，语义分析，抽象语法树
      // -{fsyntax-only,-analyze,emit-ast} only run up to the compiler.
      } else if ((PhaseArg = DAL.getLastArg(options::OPT_fsyntax_only)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT_print_supported_cpus)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT_module_file_info)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT_verify_pch)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT_rewrite_objc)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT_rewrite_legacy_objc)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT__migrate)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT__analyze)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT_emit_ast))) {
        FinalPhase = phases::Compile;
      //编译 - 中间代码生成 到达后端
      // -S only runs up to the backend.
      } else if ((PhaseArg = DAL.getLastArg(options::OPT_S)) ||
                 (PhaseArg = DAL.getLastArg(options::OPT_fsycl_device_only))) {
        FinalPhase = phases::Backend;
      //汇编 - 生成汇编代码
      // -c compilation only runs up to the assembler.
      } else if ((PhaseArg = DAL.getLastArg(options::OPT_c))) {
        FinalPhase = phases::Assemble;
      //链接 - 链接images,生成可执行文件
      // Otherwise do everything.
      } else
        FinalPhase = phases::Link;
        
      if (FinalPhaseArg)
        *FinalPhaseArg = PhaseArg;
    
      return FinalPhase;    
    }
    ```

    

 

11. 进入BuildJobs（Bind/Translate）

    clang/lib/Driver/Driver.cpp

    ```cpp
    void Driver::BuildJobs(Compilation &C) const {
        //进入14 为对应的Action 绑定对应的ToolChain以及Tool 生成Job
        BuildJobsForAction(C, A, &C.getDefaultToolChain(),
                           /*BoundArch*/ StringRef(),
                           /*AtTopLevel*/ true,
                           /*MultipleArchs*/ ArchNames.size() > 1,
                           /*LinkingOutput*/ LinkingOutput, CachedResults,
                           /*TargetDeviceOffloadKind*/ Action::OFK_None);
    }
    ```

    

12. 进入processHostLinkAction

    clang/lib/Driver/Driver.cpp

    ```cpp
    /// Processes the host linker action. This currently consists of replacing it
    /// with an offload action if there are device link objects and propagate to
    /// the host action all the offload kinds used in the current compilation. The
    /// resulting action is returned.
    Action *processHostLinkAction(Action *HostAction) {
        // Add all the dependences from the device linking actions.
        OffloadAction::DeviceDependences DDeps;
        for (auto *SB : SpecializedBuilders) {
          if (!SB->isValid())
            continue;
    
          SB->appendLinkDependences(DDeps);
        }
        ...
    }
    ```

    

13. 进入appendLinkDependences

    clang/lib/Driver/Driver.cpp

    namespace: anonymous-namespace

    ```cpp
    /// SYCL action builder. The host bitcode is passed to the device frontend
    /// and all the device linked images are passed to the host link phase.
    /// SPIR related are wrapped before added to the fat binary
    class SYCLActionBuilder final : public DeviceActionBuilder {
        void appendLinkDependences(OffloadAction::DeviceDependences &DA) override {
          // DeviceLinkerInputs holds binaries per ToolChain (TC) / bound-arch pair
          // The following will loop link and post process for each TC / bound-arch
          // to produce a final binary.
          // They will be bundled per TC before being sent to the Offload Wrapper.
            ActionList DeviceLibs;
            ActionList DeviceLibObjects;
            ActionList LinkObjects;
            auto TT = TC->getTriple();
            auto isNVPTX = TT.isNVPTX();
            auto isAMDGCN = TT.isAMDGCN();
            auto isSPIR = TT.isSPIR();
            bool isSpirvAOT = TT.getSubArch() == llvm::Triple::SPIRSubArch_fpga ||
                              TT.getSubArch() == llvm::Triple::SPIRSubArch_gen ||
                              TT.getSubArch() == llvm::Triple::SPIRSubArch_x86_64;        ...
            // The linkage actions subgraph leading to the offload wrapper.
            // [cond] Means incoming/outgoing dependence is created only when cond
            //        is true. A function of:
            //   n - target is NVPTX/AMDGCN
            //   a - SPIRV AOT compilation is requested
            //   s - device code split requested
            //   r - relocatable device code is requested
            //   f - link object output type is TY_Tempfilelist (fat archive)
            //   * - "all other cases"
            //     - no condition means output/input is "always" present
            // First symbol indicates output/input type
            //   . - single file output (TY_SPIRV, TY_LLVM_BC,...)
            //   - - TY_Tempfilelist
            //   + - TY_Tempfiletable   
    	    //
            //                   .-----------------.
            //                   |Link(LinkObjects)|
            //                   .-----------------.
            //                ----[-!rf]   [*]
            //               [-!rf]         |
            //         .-------------.      |
            //         | llvm-foreach|      |
            //         .-------------.      |
            //               [.]            |
            //                |             |
            //                |             |
            //         .---------------------------------------.
            //         |               PostLink                |
            //         .---------------------------------------.
            //                           [+*]                [+]
            //                             |                  |
            //                             |                  |
            //                             |---------         |
            //                             |        |         |
            //                             |        |         |
            //                             |      [+!rf]      |
            //                             |  .-------------. |
            //                             |  | llvm-foreach| |
            //                             |  .-------------. |
            //                             |        |         |
            //                            [+*]    [+!rf]      |
            //                      .-----------------.       |
            //                      | FileTableTform  |       |
            //                      | (extract "Code")|       |
            //                      .-----------------.       |
            //                              [-]               |-----------
            //           --------------------|                           |
            //           |                   |                           |
            //           |                   |-----------------          |
            //           |                   |                |          |
            //           |                   |               [-!rf]      |
            //           |                   |         .--------------.  |
            //           |                   |         |FileTableTform|  |
            //           |                   |         |   (merge)    |  |
            //           |                   |         .--------------.  |
            //           |                   |               [-]         |-------
            //           |                   |                |          |      |
            //           |                   |                |    ------|      |
            //           |                   |        --------|    |            |
            //          [.]                 [-*]   [-!rf]        [+!rf]         |
            //   .---------------.  .-------------------. .--------------.      |
            //   | finalizeNVPTX  | |  SPIRVTranslator  | |FileTableTform|      |
            //   | finalizeAMDGCN | |                   | |   (merge)    |      |
            //   .---------------.  .-------------------. . -------------.      |
            //          [.]             [-as]      [-!a]         |              |
            //           |                |          |           |              |
            //           |              [-s]         |           |              |
            //           |       .----------------.  |           |              |
            //           |       | BackendCompile |  |           |              |
            //           |       .----------------.  |     ------|              |
            //           |              [-s]         |     |                    |
            //           |                |          |     |                    |
            //           |              [-a]      [-!a]  [-!rf]                 |
            //           |              .--------------------.                  |
            //           -----------[-n]|   FileTableTform   |[+*]--------------|
            //                          |  (replace "Code")  |
            //                          .--------------------.
            //                                      |
            //                                    [+*]
            //         .--------------------------------------.
            //         |            OffloadWrapper            |
            //         .--------------------------------------.
            //            
        }
    }
    ```

    

14. 进入BuildJobsForAction()

    clang/lib/Driver/Driver.cpp

    ```cpp
    InputInfo Driver::BuildJobsForAction(
        Compilation &C, const Action *A, const ToolChain *TC, StringRef BoundArch,
        bool AtTopLevel, bool MultipleArchs, const char *LinkingOutput,
        std::map<std::pair<const Action *, std::string>, InputInfo> &CachedResults,
        Action::OffloadKind TargetDeviceOffloadKind) const {
      std::pair<const Action *, std::string> ActionTC = {
          A, GetTriplePlusArchString(TC, BoundArch, TargetDeviceOffloadKind)};
      auto CachedResult = CachedResults.find(ActionTC);
      if (CachedResult != CachedResults.end()) {
        return CachedResult->second;
      }
      //调用BuildJobsForActionNoCache
      InputInfo Result = BuildJobsForActionNoCache(
          C, A, TC, BoundArch, AtTopLevel, MultipleArchs, LinkingOutput,
          CachedResults, TargetDeviceOffloadKind);
      CachedResults[ActionTC] = Result;
      return Result;
    }
    
    InputInfo Driver::BuildJobsForActionNoCache(
        Compilation &C, const Action *A, const ToolChain *TC, StringRef BoundArch,
        bool AtTopLevel, bool MultipleArchs, const char *LinkingOutput,
        std::map<std::pair<const Action *, std::string>, InputInfo> &CachedResults,
        Action::OffloadKind TargetDeviceOffloadKind){
      llvm::PrettyStackTraceString CrashInfo("Building compilation jobs");
      ToolSelector TS(JA, *JATC, C, isSaveTempsEnabled(),
                      embedBitcodeInObject() && !isUsingLTO());
      //ToolSelector 获取到Tool    
      const Tool *T = TS.getTool(Inputs, CollapsedOffloadActions);  
      ...
      if (CCCPrintBindings && !CCGenDiagnostics) {
        llvm::errs() << "# \"" << T->getToolChain().getTripleString() << '"'
                     << " - \"" << T->getName() << "\", inputs: [";
        for (unsigned i = 0, e = InputInfos.size(); i != e; ++i) {
          llvm::errs() << InputInfos[i].getAsString();
          if (i + 1 != e)
            llvm::errs() << ", ";
        }
        if (UnbundlingResults.empty())
          llvm::errs() << "], output: " << Result.getAsString() << "\n";
        else {
          llvm::errs() << "], outputs: [";
          for (unsigned i = 0, e = UnbundlingResults.size(); i != e; ++i) {
            llvm::errs() << UnbundlingResults[i].getAsString();
            if (i + 1 != e)
              llvm::errs() << ", ";
          }
          llvm::errs() << "] \n";
        }
      } else {
        if (UnbundlingResults.empty())
          //构建Job返回 接收Actions 吐出Jobs  ConstructJob跳转到 17,18
          T->ConstructJob(
              C, *JA, Result, InputInfos,
              C.getArgsForToolChain(TC, BoundArch, JA->getOffloadingDeviceKind()),
              LinkingOutput);
        else
          T->ConstructJobMultipleOutputs(
              C, *JA, UnbundlingResults, InputInfos,
              C.getArgsForToolChain(TC, BoundArch, JA->getOffloadingDeviceKind()),
              LinkingOutput);
      }
      return Result;      
    }
    
    /// Check if a chain of actions can be combined and return the tool that can
    /// handle the combination of actions. The pointer to the current inputs \a
    /// Inputs and the list of offload actions \a CollapsedOffloadActions
    /// connected to collapsed actions are updated accordingly. The latter enables
    /// the caller of the selector to process them afterwards instead of just
    /// dropping them. If no suitable tool is found, null will be returned.
    const Tool *getTool(ActionList &Inputs,
                        ActionList &CollapsedOffloadAction) {
        // Attempt to combine actions. If all combining attempts failed, just return
        // the tool of the provided action. At the end we attempt to combine the
        // action with any preprocessor action it may depend on.
        //
    
        const Tool *T = combineAssembleBackendCompile(ActionChain, Inputs,
                                                      CollapsedOffloadAction);
        if (!T)
          T = combineAssembleBackend(ActionChain, Inputs, CollapsedOffloadAction);
        if (!T)
          T = combineBackendCompile(ActionChain, Inputs, CollapsedOffloadAction);
        if (!T) {
          Inputs = BaseAction->getInputs();
          //根据工具链选择Tool 跳转 15
          T = TC.SelectTool(*BaseAction);
        }
    
        combineWithPreprocessor(T, Inputs, CollapsedOffloadAction);
        return T;      
    }
    
    
    /// Struct that relates an action with the offload actions that would be
    /// collapsed with it.
    struct JobActionInfo final {
      /// The action this info refers to.
      const JobAction *JA = nullptr;
      /// The offload actions we need to take care off if this action is
      /// collapsed.
      ActionList SavedOffloadAction;
    };
    ```

    

15. 进入SelectTool()

    clang/lib/Driver/ToolChain.cpp

    ```cpp
    Tool *ToolChain::SelectTool(const JobAction &JA) const {
      if (D.IsFlangMode() && getDriver().ShouldUseFlangCompiler(JA)) return getFlang();
      if (getDriver().ShouldUseClangCompiler(JA)) return getClang();
      Action::ActionClass AC = JA.getKind();
      //使用继承汇编器的话就返回 getClangAs()
      if (AC == Action::AssembleJobClass && useIntegratedAs())
        return getClangAs();
      //根据ActionClass 获取Tool  
      return getTool(AC);
    }
    
    Tool *ToolChain::getTool(Action::ActionClass AC) const {
      switch (AC) {
      case Action::AssembleJobClass:
        return getAssemble();
    
      case Action::IfsMergeJobClass:
        return getIfsMerge();
    
      case Action::LinkJobClass:
        return getLink();
    
      case Action::StaticLibJobClass:
        return getStaticLibTool();
    
      case Action::InputClass:
      case Action::BindArchClass:
      case Action::OffloadClass:
      case Action::LipoJobClass:
      case Action::DsymutilJobClass:
      case Action::VerifyDebugInfoJobClass:
        llvm_unreachable("Invalid tool kind.");
    
      case Action::CompileJobClass:
      case Action::PrecompileJobClass:
      case Action::HeaderModulePrecompileJobClass:
      case Action::PreprocessJobClass:
      case Action::AnalyzeJobClass:
      case Action::MigrateJobClass:
      case Action::VerifyPCHJobClass:
      //获取Clang工具类        
      case Action::BackendJobClass:
        return getClang();
    
      case Action::OffloadBundlingJobClass:
      case Action::OffloadUnbundlingJobClass:
        return getOffloadBundler();
    
      case Action::OffloadWrapperJobClass:
        return getOffloadWrapper();
    
      case Action::OffloadDepsJobClass:
        return getOffloadDeps();
    
      case Action::SPIRVTranslatorJobClass:
        return getSPIRVTranslator();
    
      case Action::SPIRCheckJobClass:
        return getSPIRCheck();
    
      case Action::SYCLPostLinkJobClass:
        return getSYCLPostLink();
    
      case Action::BackendCompileJobClass:
        return getBackendCompiler();
    
      case Action::AppendFooterJobClass:
        return getAppendFooter();
    
      case Action::FileTableTformJobClass:
        return getTableTform();
      }
    
      llvm_unreachable("Invalid tool kind.");
    }
    
    Tool *ToolChain::getClang() const {
      if (!Clang)
        //跳转到tools::Clang
        Clang.reset(new tools::Clang(*this));
      return Clang.get();
    }
    
    ```

    

16. 进入Clang::Clang(const ToolChain &TC)函数

    clang/lib/Driver/ToolChains/Clang.cpp

    ```cpp
    Clang::Clang(const ToolChain &TC)
        // CAUTION! The first constructor argument ("clang") is not arbitrary,
        // as it is for other tools. Some operations on a Tool actually test
        // whether that tool is Clang based on the Tool's Name as a string.
        : Tool("clang", "clang frontend", TC) {}
    ```

    

17. 进入ConstructJob() 纯虚函数

    clang/include/clang/Driver/Tool.h

    ```cpp
    /// ConstructJob - Construct jobs to perform the action \p JA,
    /// writing to \p Output and with \p Inputs, and add the jobs to
    /// \p C.
    ///
    /// \param TCArgs - The argument list for this toolchain, with any
    /// tool chain specific translations applied.
    /// \param LinkingOutput - If this output will eventually feed the
    /// linker, then this is the final output name of the linked image.
    // 相关的实现是在Clang工具里 - 跳转18
    virtual void ConstructJob(Compilation &C, const JobAction &JA,
                                const InputInfo &Output,
                                const InputInfoList &Inputs,
                                const llvm::opt::ArgList &TCArgs,
                                const char *LinkingOutput) const = 0;
    ```

    

18. clang工具的ConstructJob()实现

    这里加入了-cc1 返回调用栈回溯到 clang/tools/driver/driver.cpp的main函数后， 重新进入clang的前端

    clang/lib/Driver/ToolChains/Clang.cpp

    ```cpp
    void Clang::ConstructJob(Compilation &C, const JobAction &JA,
                             const InputInfo &Output, const InputInfoList &Inputs,
                             const ArgList &Args, const char *LinkingOutput) const {
      ...
      //这里加入了-cc1参数，然后回溯返回 带-cc1参数重新进入clang的前端 -> 1
      // Invoke ourselves in -cc1 mode.
      //
      // FIXME: Implement custom jobs for internal actions.
      CmdArgs.push_back("-cc1");
    
      // Add the "effective" target triple.
      CmdArgs.push_back("-triple");
      CmdArgs.push_back(Args.MakeArgString(TripleStr));   
      ...    
    }
    ```

    

19. 进入ExecuteCompilation()

    此时已经机上-cc1参数准备重新进入clang前端

    clang/lib/Driver/Driver.cpp

    ```cpp
    int Driver::ExecuteCompilation(
        Compilation &C,
        SmallVectorImpl<std::pair<int, const Command *>> &FailingCommands) {
      //打印编译详细步骤  
      // Just print if -### was present.
      if (C.getArgs().hasArg(options::OPT__HASH_HASH_HASH)) {
        C.getJobs().Print(llvm::errs(), "\n", true);
        return 0;
      }
    
      // If there were errors building the compilation, quit now.
      if (Diags.hasErrorOccurred())
        return 1;
    
      // Set up response file names for each command, if necessary
      for (auto &Job : C.getJobs())
        setUpResponseFiles(C, Job);
      //执行创建的Jobs ->20 去执行
      C.ExecuteJobs(C.getJobs(), FailingCommands);  
      ...
    }
    ```

    

20. 进入ExecuteJobs()

    此时已经机上-cc1参数准备重新进入clang前端

    clang/lib/Driver/Compilation.cpp

    ```cpp
    void Compilation::ExecuteJobs(const JobList &Jobs,
                                  FailingCommandList &FailingCommands) const {
      // According to UNIX standard, driver need to continue compiling all the
      // inputs on the command line even one of them failed.
      // In all but CLMode, execute all the jobs unless the necessary inputs for the
      // job is missing due to previous failures.
      for (const auto &Job : Jobs) {
        if (!InputsOk(Job, FailingCommands))
          continue;
        const Command *FailingCommand = nullptr;
        //把-cc1加入后，即是我们要执行的过程加入到command中
        if (int Res = ExecuteCommand(Job, FailingCommand)) {
          FailingCommands.push_back(std::make_pair(Res, FailingCommand));
          // Bail as soon as one command fails in cl driver mode.
          if (TheDriver.IsCLMode())
            return;
        }
      }                              
    }
    
    int Compilation::ExecuteCommand(const Command &C,
                                    const Command *&FailingCommand) const {
      ...
      std::string Error;
      bool ExecutionFailed;
      //继续跟踪Execute函数  跳转到 21
      int Res = C.Execute(Redirects, &Error, &ExecutionFailed);
      if (PostCallback)
        PostCallback(C, Res);
      if (!Error.empty()) {
        assert(Res && "Error string set with 0 result code!");
        getDriver().Diag(diag::err_drv_command_failure) << Error;
      }
    
      if (Res)
        FailingCommand = &C;
    
      return ExecutionFailed ? 1 : Res;    
    }
    ```

    

21. 进入Execute()

    ```cpp
    int Command::Execute(ArrayRef<llvm::Optional<StringRef>> Redirects,
                         std::string *ErrMsg, bool *ExecutionFailed) const {
      PrintFileNames();
    
      SmallVector<const char *, 128> Argv;
      if (ResponseFile == nullptr) {
        Argv.push_back(Executable);
        Argv.append(Arguments.begin(), Arguments.end());
        Argv.push_back(nullptr);
      }
      ...
      auto Args = llvm::toStringRefArray(Argv.data());
      //多进程
      return llvm::sys::ExecuteAndWait(Executable, Args, Env, Redirects,
                                       /*secondsToWait*/ 0, /*memoryLimit*/ 0,
                                       ErrMsg, ExecutionFailed, &ProcStat);                     
    }
    ```

    

22. 进入int sys::ExecuteAndWait()

    完成多进程的启动和调用

    llvm/lib/Support/Program.cpp

    ```cpp
    int sys::ExecuteAndWait(StringRef Program, ArrayRef<StringRef> Args,
                            Optional<ArrayRef<StringRef>> Env,
                            ArrayRef<Optional<StringRef>> Redirects,
                            unsigned SecondsToWait, unsigned MemoryLimit,
                            std::string *ErrMsg, bool *ExecutionFailed,
                            Optional<ProcessStatistics> *ProcStat,
                            BitVector *AffinityMask) {
      assert(Redirects.empty() || Redirects.size() == 3);
      //进程信息
      ///// @brief This struct encapsulates information about a process.
      ProcessInfo PI;
      //进入Execute创建多线程执行 -> 23
      if (Execute(PI, Program, Args, Env, Redirects, MemoryLimit, ErrMsg,
                  AffinityMask)) {
        if (ExecutionFailed)
          *ExecutionFailed = false;
        //等待子进程执行结果，如果SecondsToWait==0,则阻塞等待  ->24    
        ProcessInfo Result =
            Wait(PI, SecondsToWait, /*WaitUntilTerminates=*/SecondsToWait == 0,
                 ErrMsg, ProcStat);
        return Result.ReturnCode;
      }
    
      if (ExecutionFailed)
        *ExecutionFailed = true;
    
      return -1;
    }
    ```

    

23. 进入Execute()执行

    在执行Command的时候，发现工具链是Clang，则会走入到ExecuteCC1Tool中。

    ExecuteCC1Tool执行会进入到cc1_main.

    llvm/lib/Support/Unix/Program.inc

    ```cpp
    static bool Execute(ProcessInfo &PI, StringRef Program,
                        ArrayRef<StringRef> Args, Optional<ArrayRef<StringRef>> Env,
                        ArrayRef<Optional<StringRef>> Redirects,
                        unsigned MemoryLimit, std::string *ErrMsg,
                        BitVector *AffinityMask) {
    ...
      // Create a child process.
      int child = fork();
      switch (child) {
        // An error occurred:  Return to the caller.
        case -1:
          MakeErrMsg(ErrMsg, "Couldn't fork");
          return false;
    
        // Child process: Execute the program.
        //子进程执行Program
        case 0: {
          // Redirect file descriptors...
          if (!Redirects.empty()) {
            // Redirect stdin
            if (RedirectIO(Redirects[0], 0, ErrMsg)) { return false; }
            // Redirect stdout
            if (RedirectIO(Redirects[1], 1, ErrMsg)) { return false; }
            if (Redirects[1] && Redirects[2] && *Redirects[1] == *Redirects[2]) {
              // If stdout and stderr should go to the same place, redirect stderr
              // to the FD already open for stdout.
              if (-1 == dup2(1,2)) {
                MakeErrMsg(ErrMsg, "Can't redirect stderr to stdout");
                return false;
              }
            } else {
              // Just redirect stderr
              if (RedirectIO(Redirects[2], 2, ErrMsg)) { return false; }
            }
          }
    
          // Set memory limits
          if (MemoryLimit!=0) {
            SetMemoryLimits(MemoryLimit);
          }
    
          // Execute!
          std::string PathStr = std::string(Program);
          //有环境变量
          if (Envp != nullptr)
            //内核级系统调用，其余execl/execle/execlp/execv/execvp都是调用execve的库函数
            //用来执行参数filename字符串所代表的文件路径，第二个参数时利用数组指针来传递给执行文件，并且需要以空指针(NULL)结束，最后一个参数则为传递给执行文件的新环境变量数组
            //从这里又回到 1
            execve(PathStr.c_str(), const_cast<char **>(Argv),
                   const_cast<char **>(Envp));
          else
            //execv()用来执行参数path字符串做代表文件路径，
            execv(PathStr.c_str(), const_cast<char **>(Argv));
          // If the execve() failed, we should exit. Follow Unix protocol and
          // return 127 if the executable was not found, and 126 otherwise.
          // Use _exit rather than exit so that atexit functions and static
          // object destructors cloned from the parent process aren't
          // redundantly run, and so that any data buffered in stdio buffers
          // cloned from the parent aren't redundantly written out.
          _exit(errno == ENOENT ? 127 : 126);
        }
        //父进程跳出，记录PI的子进程编号，返回 ->22 去等待子进程执行结果
        // Parent process: Break out of the switch to do our processing.
        default:
          break;
      }
    
      PI.Pid = child;
      PI.Process = child;
    
      return true;    
    }
    ```

    

24. 进入Wait()

    llvm/lib/Support/Unix/Program.inc

    ```cpp
    ProcessInfo llvm::sys::Wait(const ProcessInfo &PI, unsigned SecondsToWait,
                                bool WaitUntilTerminates, std::string *ErrMsg,
                                Optional<ProcessStatistics> *ProcStat) {
      ...    
      // Parent process: Wait for the child process to terminate.
      int status;
      ProcessInfo WaitResult;
      rusage Info;
      if (ProcStat)
        ProcStat->reset();
    
      do {
        WaitResult.Pid = sys::wait4(ChildPid, &status, WaitPidOptions, &Info);
      } while (WaitUntilTerminates && WaitResult.Pid == -1 && errno == EINTR);
    
      if (WaitResult.Pid != PI.Pid) {
        if (WaitResult.Pid == 0) {
          // Non-blocking wait.
          return WaitResult;
        } else {
          //阻塞等待
          if (SecondsToWait && errno == EINTR) {
            // Kill the child.
            kill(PI.Pid, SIGKILL);
    
            // Turn off the alarm and restore the signal handler
            alarm(0);
            sigaction(SIGALRM, &Old, nullptr);
    
            // Wait for child to die
            // FIXME This could grab some other child process out from another
            // waiting thread and then leave a zombie anyway.
            if (wait(&status) != ChildPid)
              MakeErrMsg(ErrMsg, "Child timed out but wouldn't die");
            else
              MakeErrMsg(ErrMsg, "Child timed out", 0);
    
            WaitResult.ReturnCode = -2; // Timeout detected
            return WaitResult;
          } else if (errno != EINTR) {
            MakeErrMsg(ErrMsg, "Error waiting for child process");
            WaitResult.ReturnCode = -1;
            return WaitResult;
          }
        }
      }
      ...
    }    
    ```

    

25. 

    

    

    

 