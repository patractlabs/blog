# Support wasmtime in sandbox

Just supported [wasmtime][wasmtime] in the sandbox of [europa][europa] these days, we 
can execute ink! contracts using JIT and trace the panicking line numbers of the rust 
code with DWARF for now.


## 1. The arch of sandbox

The `sandbox` we are talking here is `primitives/sandbox` which executes ink! contract
in `pallet-contracts`, it abstracts a WASM layer that can adapt different wasm executors.

```rust
/// Sandbox liner memory
struct Memory { ... }

/// WASM imports
struct EnvironmentDefinitionBuilder<T> { ... }

/// WASM module instance
struct Instance { ... }
```

### 1.1 Memory

The host functions defined in `pallet-contracts` use `(ptr: u32, len: u32,)` as arguments
since WASM only can accept integer.

The process is like:

+ (pallet-contract): Alloc memory from the liner memory of WASM
+ (pallet-contract): Passing pointers into wasm interface
+ (wasm-executor):   Get porinters and executue functions
+ (wasm-executor):   Return result to pallet-contract
+ (pallet-contract): Read result from liner memory through pointers


### 1.2 Environment

The environment in sandbox includes `Host Functions` and `Memory`, both memory and functions
could be externed outside WASM module. It sets up the interaction between native machine and 
wasm module.


### 1.3 Instance

WASM module with human friendly interfaces comes to instance, it used for invoking functions
in the sandbox.


## 2. Wasmtime

There are not a lot of trouble adapting wasmtime into the sandbox, passing generic types to 
the host function may be the one could be mentioned.


```rust
pub fn wrap_fn<T>(store: &Store, state: usize, f: usize, sig: FunctionType) -> Func {
	let func = move |_: Caller<'_>, args: &[Val], results: &mut [Val]| {
		let mut inner_args = vec![];
		for arg in args {
			if let Some(arg) = from_val(arg.clone()) {
				inner_args.push(arg);
			} else {
				return Err(Trap::new("Could not wrap host function"));
			}
		}

		// HACK the LIFETIME
		//
		// # Safety
		//
		// Runtime only run for one call.
		let state: &mut T = unsafe { mem::transmute(state) };
		let func: HostFuncType<T> = unsafe { mem::transmute(f) };
		match func(state, &inner_args) {
			Ok(ret) => {
				if let Some(ret) = from_ret_val(ret) {
					results[0] = ret;
				}
				Ok(())
			}
			Err(_) => Err(Trap::new("Could not wrap host function")),
		}
	};
	Func::new(store, wasmtime_sig(sig), func)
}
```

Likewise, we pass pointers to the `Func` struct of wasmtime with `unsafe` code that we can define
our host functions.


## 3. DWARF

The support of DWARF in wasm is still in an early stage for now, which hasn't been written into the
SPEC, but an document [DWARF for WebAssembly][dwarf] has introduced it.

As a result, we use the `DWARF` implementation in wasmtime directly now, it could be enable with 
envrionment `WASM_BACKTRACE_DETAILS=1` as how it works in wasmtime.

[europa]: https://github.com/patractlabs/europa
[wasmtime]: https://github.com/bytecodealliance/wasmtime
[dwarf]: https://yurydelendik.github.io/webassembly-dwarf/
