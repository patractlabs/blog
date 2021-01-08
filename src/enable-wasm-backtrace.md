# Enable WASM backtrace

Just implemented WASM backtrace for the contract system of substrate these days, here is a panicked flipper to show how it works:

```rust
#![cfg_attr(not(feature = "std"), no_std)]

use ink_lang as ink;

#[ink::contract]
mod flipper {
    #[ink(storage)]
    pub struct Flipper {
        value: bool,
    }

    impl Flipper {
        #[ink(constructor)]
        pub fn new(init_value: bool) -> Self {
            panic!("Flip failed!");
            Self { value: init_value }
        }

        #[ink(constructor)]
        pub fn default() -> Self {
            Self::new(Default::default())
        }

        // This message will emit `ContractTrapped` everytime 
        // people call it from node.
        #[ink(message)]
        pub fn flip(&mut self) {
            self.value = !self.value;
        }

        #[ink(message)]
        pub fn get(&self) -> bool {
            self.value
        }
    }
}
```

#### Before WASM Backtrace

Let's see how the panicked flipper panicing in the [pallet-contracts/src/wasm/runtime.rs#L378][pallet-contract-err].

```txt
Trap {
    kind: DummyHostError,
}
```

#### After WASM Backtrace

And the panic could be like:

```txt
Trap {
    kind: Unreachable,
    wasm_trace: [
        "core::panicking::panic[130]",
        "tt::tt::_::<impl tt::tt::Tt>::new[162]",
        "tt::tt::_::<impl tt::tt::Tt>::default[163]",
        ::tt::_::__ink_Constr<[(); 3792844650]> as ink_lang::traits::Constructor>::CALLABLE::{{closure}}[46]",
        ::ops::function::FnOnce::call_once[45]",
        ::tt::_::_::__ink_ConstructorDispatchEnum as ink_lang::dispatcher::Execute>::execute::{{closure}}[150]",
        _lang::dispatcher::execute_constructor[149]",
        "<tt::tt::_::_::__ink_ConstructorDispatchEnum as ink_lang::dispatcher::Execute>::execute[159]",
        "tt::tt::_::<impl ink_lang::contract::DispatchUsingMode for tt::tt::Tt>::dispatch_using_mode[156]",
        "deploy[155]",
        "<unknown>[596]",
    ],
}
```

Let's track out how to reach the effect above!


## 1. Submit WASM files contains debug info

* PR:     [paritytech/cargo-contract#131](https://github.com/paritytech/cargo-contract/pull/131)
* Source: [patractlabs/cargo-contract](https://github.com/patractlabs/cargo-contract)

Frist of all, we have to submit **wasm files which contain the debug info** that the on-chain
side can parse and get the panic errors.

Currently, [parity/cargo-contract](https://github.com/patractlabs/cargo-contract) trims debug
info while building contracts, we can get them back with the following steps.

### 1.1 Keep name section from striping

The name section has been striped at [cargo-contract/cmd/build.rs#L247][cargo-contract/strip].


### 1.2 Keep debug info from `wasm-opt`

`cargo-contract` passes `debug_info: false` to `wasm-opt`, so the debug-info will be always optimized 
when we run `wasm-opt`, the code is here: [cargo-contract/cmd/build.rs#L267][cargo-contract/wasm-opt].


### 1.3 Keep debug info from `rustc`

`cargo-contract` executes realease build by default, here we can enable debug build or modify 
the opt-level flag of `rustc` to keep debug infos at [cargo-contract/cmd/build.rs#L137][cargo-contract/release].


## 2. Save debug info from the scraper of `pallet-contracts`

What happends to the `pallet-contracts` while we calling a contract?

* **Get the WASM binary from storage**
* **Inject gas meter to the contract**
* Inject stack height to the contract
* Put the contract into `sp-sandbox` and execute it
* Get the result from `sp-sandbox` and return the result to us


### 2.1 Store name section while building WASM module

* PR: https://github.com/paritytech/parity-wasm/pull/300

`pallet-contracts` builds WASM modules from storage and drops custom sections by default,
here we should get them back.


### 2.2 Update function indices in name section while injecting gas meter

* PR:      [paritytech/wasm-utils#146](https://github.com/paritytech/wasm-utils/pull/146)
* Source:  [patractlabs/wasm-utils#146](https://github.com/patractlabs/wasm-utils)

`pallet-contracts` reorders the indcies of functions in our WASM contracts after injecting gas memter
to the WASM contracts at [paritytech/wasm-utils/gas/mod.rs#L467][wasm-utils/gas], this will mess up the
function indecies in name section that we can not get the correct backtraces.


## 3. Impelment WASM backtrace to WASMI

* Source:  [patractlabs/wasmi](https://github.com/patractlabs/wasmi)

Remember the last two steps in chapter 2, the core part of enabling WASM backtrace to pallet-contract
is making `wasmi` support backtrace.

The process of executing a function in the interpreter of wasmi is like:

* Invoke the target function
* call and call and call over again
  * Panic if the process breaks.


### 3.1 Add function info field to `FuncRef`

`FuncRef` is the 'function' wasmi interpreter calling directly, so we need to embed the debug info
into the `FuncRef` as the first time, source at [wasmi/func.rs#L26][wasmi/func].

```rust
//! patractlabs/wasmi/src/func.rs#L26
/// ...
pub struct FuncRef {
    /// ...
    /// Function name and index for debug
    info: Option<(usize, String)>,
}
```

### 3.2 Set function info using name section while parsing WASM modules

We alread have the `info` field in `FuncRef`, now we need to fill this field using name setciton 
while parsing WASM modules, source at [wasmi/module#L343][wasmi/module].

```rust
//! wasmi/src/module.rs#L343
// ...
if has_name_section {
     if let Some(name) = function_names.get((index + import_count) as u32) {
         func_instance = func_instance.set_info(index, name.to_string());
     } else {
         func_instance = func_instance.set_info(index, "<unknown>".to_string());
     }
}
// ...
```


### 3.3 Make the interpreter support `trace`

However, we don't need to get infos of every functions but the panicked series in the stack of the
interpreter, source at [wasmi/runner.rs#L1491][wasmi/trace].

```rust
//! wasmi/src/runner.rs#L1491
/// Get the functions of current the stack
pub fn trace(&self) -> Vec<Option<&(usize, String)>> {
    self.buf.iter().map(|f| f.info()).collect::<Vec<_>>()
}
```

### 3.4 Gather debug infos when program breaks

Just gather backtraces when we invoke function failed, source at [wasmi/func.rs#L170][wasmi/invoke].

```rust
//! wasmi/src/func.rs#L170
// ...

let res = interpreter.start_execution(externals);
if let Err(trap) = res {
let mut stack = interpreter
    .trace()
    .iter()
    .map(|n| {
        if let Some(info) = n {
            format!("{:#}[{}]", rustc_demangle::demangle(&info.1), info.0)
        } else {
            "<unknown>".to_string()
        }
    })
    .collect::<Vec<_>>();

// Append the panicing trap
stack.append(&mut trap.wasm_trace().clone());
stack.reverse();

// Embed this info into the trap
Err(trap.set_wasm_trace(stack))

// ...
```


### 4. Assembling the modules

The last thing is assembling the modules above, if you compose like above, you'll got a [europa][europa].


[pallet-contract-err]: https://github.com/paritytech/substrate/blob/885c1f17cb6fcb542d7977ea3bd2136b9466c921/frame/contracts/src/wasm/runtime.rs#L378
[cargo-contract/strip]: https://github.com/paritytech/cargo-contract/blob/71525f9ec5f21e6113a614c2fb4d1eb5e62ebf8b/src/cmd/build.rs#L247
[cargo-contract/wasm-opt]: https://github.com/paritytech/cargo-contract/blob/71525f9ec5f21e6113a614c2fb4d1eb5e62ebf8b/src/cmd/build.rs#L267
[cargo-contract/release]: https://github.com/paritytech/cargo-contract/blob/71525f9ec5f21e6113a614c2fb4d1eb5e62ebf8b/src/cmd/build.rs#L137
[wasm-utils/gas]: https://github.com/paritytech/wasm-utils/blob/d9432bafa9321f8e0e5b8a143f1ed858dbbbe272/src/gas/mod.rs#L467
[wasmi/func]: https://github.com/patractlabs/wasmi/blob/v0.6.2/src/func.rs#L26
[wasmi/trace]: https://github.com/patractlabs/wasmi/blob/7a6feaea70f47aa6e62f097fb0d9a4ea0ce7d1fc/src/runner.rs#L1491
[wasmi/module]: https://github.com/patractlabs/wasmi/blob/7a6feaea70f47aa6e62f097fb0d9a4ea0ce7d1fc/src/runner.rs#L1491
[wasmi/invoke]: https://github.com/patractlabs/wasmi/blob/7a6feaea70f47aa6e62f097fb0d9a4ea0ce7d1fc/src/func.rs#L170
[wasmi/module]: https://github.com/patractlabs/wasmi/blob/7a6feaea70f47aa6e62f097fb0d9a4ea0ce7d1fc/src/module.rs#L343
[europa]: https://github.com/patractlabs/europa
