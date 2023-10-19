# Code coverage for WebAssembly

This repository demonstrates how to generate code coverage report for WebAssembly programs written in rust, especially smart contracts in blockchain protocols. As of 18 October 2023 generation of code coverage report was not possible in any of blockchain protocols using WebAssembly VM. The goal of this repository is to demonstrate how every protocol using WebAssembly VM can implement code coverage functionallity in their ecosystem. Technique demonstrated in this repository is not only limited to blockchain protocols, it should be possible to use it with any project using WASM VM.

### Step 1: Adding code coverage instrumentation to WASM binary

There's a project called [minicov](https://github.com/Amanieu/minicov/) which easily allows to add LLVM instrumentation coverage to any rust project. So we need to add minicov to our dependencies:
```rust
[dependencies]
minicov = "0.3"
```
and then in our code add the following function:
```rust
#[no_mangle]
unsafe extern "C" fn dump_coverage() {
    let mut coverage = vec![];
    minicov::capture_coverage(&mut coverage).unwrap();
    ScryptoVmV1Api::dump_coverage(coverage); // a function which will save coverage data to file
    //println!("{:?}, coverage); // eventually we can just print it and copy it manually
}
```
and then we compile our project with the RUSTFLAGS set to:
```bash
RUSTFLAGS="-Cinstrument-coverage -Zno-profiler-runtime --emit=llvm-ir"
```
In the `minicov` documentation there's no mention of `--emit=llvm-ir`, however we'll need it later.

There is one challange with this code, we need a function in VM to save data from `minicov::capture_coverage` to a file. By default, almost no blockchain VM supports such functinallity so it needs to be added. In this case, I've added new native function called `ScryptoVmV1Api::dump_coverage`, which has the following implementation:
```rust
fn dump_coverage(&mut self, data: Vec<u8>) -> Result<(), RuntimeError> {        
    // blueprint_id is the name of project
    let blueprint_id = self.current_actor()
        .blueprint_id()
        .ok_or(RuntimeError::SystemError(SystemError::NoBlueprintId))?;

    if let Some(dir) = env::var_os("COVERAGE_DIRECTORY") {
        let mut file_path = Path::new(&dir).to_path_buf();
        file_path.push(blueprint_id.blueprint_name);
        // Check if the directory exists, if not create it
        if !file_path.exists() {
            fs::create_dir(&file_path).unwrap();
        }
        // file name is hash of its data, so there's no chance of collison
        let file_name = hash(&data);
        let file_name: String = file_name.0[..16]
            .iter()
            .map(|byte| format!("{:02x}", byte))
            .collect();
        file_path.push(format!("{}.profraw", file_name));
        let mut file = File::create(file_path).unwrap();
        file.write_all(&data).unwrap();
    }
    Ok(())
}
```

### Step 2: Automatically calling dump_coverage

So we have a function which allows us to dump coverage data, now we just need to call it every time after execution of our code. Adding it manually to each function is not a solution, so the VM must be modified that it calls `dump_coverage` every time after execution is finished. This step is a platform specific and may not be easy, especially when execution fails (eg. due to panic in the code). 

Here's an example how it can be implemented in [radix wasmi](https://github.com/radixdlt/radixdlt-scrypto/blob/main/radix-engine/src/vm/wasm/wasmi.rs) VM implementation: the function responsible for calling WASM function is called `invoke_export` and it looks like this:
```rust
impl WasmInstance for WasmiInstance {
    fn invoke_export<'r>(
        &mut self,
        func_name: &str,
        args: Vec<Buffer>,
        runtime: &mut Box<dyn WasmRuntime + 'r>,
    ) -> Result<Vec<u8>, InvokeError<WasmRuntimeError>> {
        ...
        let func = self.get_export_func(func_name).unwrap();
        let input: Vec<Value> = args
            .into_iter()
            .map(|buffer| Value::I64(buffer.as_i64()))
            .collect();
        let mut ret = [Value::I64(0)];

        let _result = func
            .call(self.store.as_context_mut(), &input, &mut ret)
            .map_err(|e| {
                let err: InvokeError<WasmRuntimeError> = e.into();
                err
            })?;

        match i64::try_from(ret[0]) {
            Ok(ret) => read_slice(
                self.store.as_context_mut(),
                self.memory,
                Slice::transmute_i64(ret),
            ),
            _ => Err(InvokeError::SelfError(WasmRuntimeError::InvalidWasmPointer)),
        }
    }
}
```

It's quite simple, it resolves export, prepares input and then parse output from result. Here's how we can modify it to call `dump_coverage` after each execution:
```rust
fn invoke_export<'r>(
    &mut self,
    func_name: &str,
    args: Vec<Buffer>,
    runtime: &mut Box<dyn WasmRuntime + 'r>,
) -> Result<Vec<u8>, InvokeError<WasmRuntimeError>> {
    ...
    let func = self.get_export_func(func_name).unwrap();
    let input: Vec<Value> = args
        .into_iter()
        .map(|buffer| Value::I64(buffer.as_i64()))
        .collect();
    let mut ret = [Value::I64(0)];

    let call_result = func
        .call(self.store.as_context_mut(), &input, &mut ret)
        .map_err(|e| {
            let err: InvokeError<WasmRuntimeError> = e.into();
            err
        });

    // store result of execution to be returned after call to dump_coverage
    let result = match call_result {
        Ok(_) => { 
            match i64::try_from(ret[0]) {
                Ok(ret) => read_slice(
                    self.store.as_context_mut(),
                    self.memory,
                    Slice::transmute_i64(ret),
                ),
                _ => Err(InvokeError::SelfError(WasmRuntimeError::InvalidWasmPointer)),
            }
        },
        Err(err) => {
            Err(err)
        }
    };

    // now it checks if there's dump_coverage function in the code
    if let Ok(func) = self.get_export_func("dump_coverage") {
        // the code contains dump_coverage, we call it with no arguments
        match func.call(self.store.as_context_mut(), &vec![], &mut vec![]).map_err(|e| {
            let err: InvokeError<WasmRuntimeError> = e.into();
            err
        }) {
            Err(InvokeError::SelfError(WasmRuntimeError::NotImplemented)) => {
                // code is not really executed, this error can be ignored
            }
            Err(err) => {
                panic!("dump_coverage failed with error {err:?}");
            }
            Ok(_) => {}
        };
    }

    // return the result of the call
    result
}
```

Now we have everything we need to generate `.profraw` files with coverage data, we just need to run our code.

### Step 3: Converting .profraw into .profdata

After running our code, we'll end up with ore or more `.profraw` files:
```bash
-rw-r--r-- 1 vscode vscode 2.1M Oct 18 11:04 33fdb846dc534782ca5ec51b6156603c.profraw
-rw-r--r-- 1 vscode vscode 2.1M Oct 18 11:03 5e526d12cde98071cc2c2227507a6441.profraw
-rw-r--r-- 1 vscode vscode 2.1M Oct 18 11:04 ab01a1ddb3854a6313fc93118d64aa60.profraw
```
Now we need to convert it into single `.profdata` file which can be done with the following command:
```bash
llvm-profdata merge -sparse *.profraw -o coverage.profdata
```

We should make sure that we're using the same major version of `llvm-profdata` as our rust version. It can be checked with `rustc --version --verbose`:
```bash
rustc 1.75.0-nightly (09df6108c 2023-10-17)
binary: rustc
commit-hash: 09df6108c84fdec400043d99d9ee232336fd5a9f
commit-date: 2023-10-17
host: x86_64-unknown-linux-gnu
release: 1.75.0-nightly
LLVM version: 17.0.2
```
And we can check `llvm` version with `llvm-config --version`, which in my case is 14.0.6 which doesn't match version 17.0.2 used by rustc. Because of that I need to [install llvm version 17](https://apt.llvm.org/) and use `llvm-profdata-17` command instead of `llvm-profdata`. So in my case, the correct command is:
```bash
llvm-profdata-17 merge -sparse *.profraw -o coverage.profdata
```

This part with llvm version match is a bit tricky because there's no simple way to install the same version of llvm as rust is using so it may be challenging to automate the whole process with a single command.

### Step 4: Getting coverage report

Now with the `coverage.profdata` we're ready to generate coverate report. First, let's try to use the following command:
```bash
llvm-cov-17 show --instr-profile=coverage.profdata our_binary.wasm
```
Unfortunetlly, it's not going to work because `our_binary.wasm` doesn't have `__llvm_covmap` section required by `llvm-cov` to generate coverage report.
```bash
$ llvm-cov-17 show --instr-profile=coverage.profdata our_binary.wasm
error: Failed to load coverage: 'our_binary.wasm': No coverage data found
```

Fortunately, there is a way to make it work by doing few extra steps. First of all we need to find Intermediate Representation file which was generated by adding `--emit=llvm-ir` to RUSTFLAGS, in this case it is `our_binary.ll` which was located in my case in `target/wasm32-unknown-unknown/release/deps` directory. We can use this file to compile our program (only object files, without linking) for x86_64 architecture instead of WASM and use it to generate code coverage report. We can do it with a command:
```bash
clang-17 our_binary.ll -Wno-override-module -c
```
This command will create `our_binary.o`` which will work with llvm-cov, now we can use it to generate code coverage report:
```bash
llvm-cov-17 show --instr-profile=coverage.profdata our_binary.o --format=html -output-dir=coverage/
```
It will correctly generate code coverage report an save it `coverage` directory as html.

However, this approach is not going to work if the project is using instruciton only available in WASM like, `memory.size`, `memory.grow`. In that case we need to do an extra step. Fortunetlly, instructions in `our_binary.ll` are not needed to generate code coverage report, so we can just remove all of them. Functions in inte looks like this:
```
; Function Attrs: minsize nounwind optsize
define dso_local void @dump_coverage() unnamed_addr #0 {
start:
  %_6 = alloca %"alloc::vec::Vec<u8>", align 4
  %coverage = alloca %"alloc::vec::Vec<u8>", align 4
  %0 = atomicrmw add ptr @__profc_dump_coverage, i64 1 monotonic, align 8
  call void @llvm.lifetime.start.p0(i64 12, ptr nonnull %coverage)
  ...
  call void @llvm.lifetime.end.p0(i64 12, ptr nonnull %_6)
  call void @llvm.lifetime.end.p0(i64 12, ptr nonnull %coverage)
  ret void
}
```
So to remove the instruction for each defined function we can change it to:
```
; Function Attrs: minsize nounwind optsize
define dso_local void @dump_coverage() unnamed_addr #0 {
start:
  unreachable
}
```

Doing it manually would be impossible, but fortunettly we can use a regular expression to modify all of them:
```bash
perl -i -p0e 's/(^define[^\n]*\n).*?^}\s*$/$1start:\n  unreachable\n}\n/gms' our_binary.ll
```
And then we can call
```bash
clang-17 our_binary.ll -Wno-override-module -c
llvm-cov-17 show --instr-profile=coverage.profdata our_binary.o --format=html -output-dir=coverage/
```
to generate code coverage report.

## Example implementation

Within next few days, we are going to add example implementation of code coverage for [radixdlt-scrypto](https://github.com/radixdlt/radixdlt-scrypto/) and [nearcore](https://github.com/near/nearcore).

## Remarks

I am aware this solution is not perfect and is using few hacks, but it is working. There is a batter way, just adding a support for WASM files to `llvm-cov`, however it is way harder to implementent and would take much more time. Because of that, I recommend to use this approach.
