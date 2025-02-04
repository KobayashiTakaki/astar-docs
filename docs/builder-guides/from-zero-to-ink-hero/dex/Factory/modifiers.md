---
sidebar_position: 3
---

# Custom Modifier

in factory contract prior entering **setFeeTo** and **setFeeToSetter** there is a [check](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Factory.sol#L41) that caller is `feeToSetter`.
Let's create a custom modifier for it.

## `only_fee_setter`

In *.logics/impls/factory/factory.rs* import `modifier_definition` and `modifiers`:
```rust
use openbrush::{
    modifier_definition,
    modifiers,
    traits::{
        AccountId,
        Storage,
        ZERO_ADDRESS,
    },
};
```

Let's define the generic modifier below the `impl` block. Some rules generic type parameters:
- If it should use Storage structs - it should take it as type parameter
- It should have the same return type - `Result<R, E>` where E is wrapped in From Trait

In the body of modifier we ensure that caller is the address stored as `fee_to_setter` and return an Error otherwise:
```rust
#[modifier_definition]
pub fn only_fee_setter<T, F, R, E>(instance: &mut T, body: F) -> Result<R, E>
    where
        T: Storage<data::Data>,
        F: FnOnce(&mut T) -> Result<R, E>,
        E: From<FactoryError>,
{
    if instance.data().fee_to_setter != T::env().caller() {
        return Err(From::from(FactoryError::CallerIsNotFeeSetter))
    }
    body(instance)
}
```

Add the modifier on top of **set_fee_to** and **set_fee_to_setter**:
```rust
#[modifiers(only_fee_setter)]
fn set_fee_to(&mut self, fee_to: AccountId) -> Result<(), FactoryError> {
    self.data::<data::Data>().fee_to = fee_to;
    Ok(())
}

#[modifiers(only_fee_setter)]
fn set_fee_to_setter(&mut self, fee_to_setter: AccountId) -> Result<(), FactoryError> {
    self.data::<data::Data>().fee_to_setter = fee_to_setter;
    Ok(())
}
```

Add `CallerIsNotFeeSetter` variant to `FactoryError`:
```rust
#[derive(Debug, PartialEq, Eq, scale::Encode, scale::Decode)]
#[cfg_attr(feature = "std", derive(scale_info::TypeInfo))]
pub enum FactoryError {
    PairError(PairError),
    CallerIsNotFeeSetter,
    ZeroAddress,
    IdenticalAddresses,
    PairInstantiationFailed,
}
```

And that's it! Check your Factory contract with (to run in contract folder):
```console
cargo contract build
```
It should now look like this [branch](https://github.com/AstarNetwork/wasm-tutorial-dex/tree/tutorial/factory_modifiers).
