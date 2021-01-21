# Support wasmtime in sandbox

Due to the performance of wasm JIT, we have supported wasmtime in `sp-sandbox` which called `ep-sandbox` in europa.

Why and what is the sandbox?

It is a bench of wasm interfaces that can connect wasm executor with the `pallet-contracts`.


## 1. Memory



## 2. Functions

### 2.1 WASM Functions

The main usage of sandbox is executing `WASM functions`, likewise, our contracts are compiled into wasm,
calling contracts, there are WASM functions running in the sandbox.

### 2.2 Host Functions

For calcuating WASM types without using WASM functions, there are `host functions` which uses wasm types
as parameters and results, running outside of the wasm executor.
