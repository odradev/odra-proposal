# Odra Proposal
Odra is high-level smart contract framework for Rust, which encourages rapid development and clean, pragmatic design. Built by experienced developers, it takes care of much of the hassle of smart contract development, enabling you to focus on writing your dapp without the need to reinvent the wheel. It's free and open source.

## Universal design.
Odra's goal is to become the go-to smart contract framework for all WebAssembly-based blockchains. We have examined multiple existing platforms and designed Odra to be reusable across many of them: Ethereum's eWASM, Polkadot's contract pallete, CosmWasm, Near, Solana, and Casper. A smart contract written using Odra can be executed on all integrated systems. We can do it by abstracting over core concepts that all the above systems are built around. These are types system, storage, entry points, execution context, and testing environment. We believe it will bring standardization to the development of Rust-based smart contracts and enable code reusability we have not yet seen in this ecosystem. 

## Casper Blockchain
The Casper blockchain became a fully operational blockchain by now. Given its current technical achievements, plans, and strong tech, we see Casper as a great first platform to integrate with Odra, before we move to other systems.

## What problem do we want to solve?
We have learned that writing smart contracts for Casper can be much easier. Abstracting away a low-level code and enforcing well-tested design patterns for writing smart contracts would unleash the developers' potential and let them focus only on what's essential while keeping the code clean, safe, and maintainable. Furthermore, it simplifies the code, reduces its amount, and makes it more readable. Those things allow the project to finish earlier and lower the development costs. The right tool for writing smart contracts will attract new developers and companies to bootstrap their projects on Casper. We also target Solidity developers by adapting concepts and best practices from this project.

## Other smart contract platforms.
It is worth exploring other smart contract platforms' tools and learning from them. 

### Ethereum
In 2017 we observed an explosion of Ethereum based projects. It took two years after launch to find a proper set of tools for writing smart contracts. Not many remember another smart contract language that existed at the time - the Serpent. It failed at becoming a standard due to its lack of tooling for scaling projects and too many low-level features. The community has learned the lessons and settled at developing Solidity as the primary smart contract language. The language with a JavaScript-like syntax and high-level abstractions made it easy to develop and understand. It set a standard for defining smart contracts, which many will copy. It is believed that Solidity is the single most crucial tool that helped Ethereum become so popular. Solidity is designed for EVM, but we can learn significantly from it. Its features and simplicity lead to the development of OpenZeppelin, the most reusable and extendable library in the ecosystem. Providing core elements like tokens and access control allowed developers to build much more extensive projects on top of it. However, it is still hard to test Solidity code.

### Polkadot
Polkadot's contract pallet is one of the most advanced WebAssembly-based smart contact platforms. They have developed an eDSL called ink!. It is an attempt to allow developers to write Solidity-style smart contracts. It is an excellent example of how to use Rust macros. What we like is how testable the ink's code is. Unfortunately, the framework is not portable to Casper.

## What do we want to offer?
To win developers' hearts, we want to offer:
1. Rust-friendly way of building smart contracts. 
2. Solidity-like smart contract definition patterns.
3. Rich standard library of popular contracts and modules.
4. Modularized code, easy to share between projects.
5. Easy way of testing contracts and unit test smaller modules.
6. Code structure that is suitable for writing small and large projects.
7. Ability to debug smart contracts via Rust debugger.
8. Complete documentation and tutorials.

# Odra 
To build a contract using `odra`, developers need to understand three basic concepts: parts, modules, contracts, and tests.

## Parts
Parts are fundamental elements for storage interaction.

### Variable
The abstraction that allows to write and retrieve a single value to/from storage is `Variable`. It automatically returns the default value if nothing lives under a given key.

```rust
let mut number: Variable<u32> = Variable::new("number");
assert_eq!(number.get(), 0);
number.set(10);
assert_eq!(number.get(), 10);
```

### Mappings
The `Mapping` can be used for handling key-value storage. Its name is borrowed from Solidity. `Mapping` is built on top of Casper's dictionary. It solves dictionary limitations of a need for caching dictionary seed and allows removal of the elements. Also, it automatically returns the default value if nothing lives under a given key.

```rust
let mut map: Mapping<u32, bool> = Mapping::new("map");
assert_eq!(map.get(10), false);
map.set(10, true);
assert_eq!(map.get(10), true);
map.delete(10);
assert_eq!(map.get(10), false);
```

### Events
Casper doesn't provide events as we know them from other platforms. But it gives all the building blocks that allow to implement them. We propose to use Structs annotated with `#[odra::event]` to define events and push them out using the `emit` method. 

```rust
#[odra::event]
struct NewEntry {
    pub name: String,
    pub value: u32
}

emit(NewEntry {
    name: String::from("number"),
    value: 10
});
```

### Modules
In `odra`, a module is a reusable element composed of other modules, variables, or mappings.

Below is the example implementation of a simple `Counter` module.

```rust
#[odra::module]
struct Counter {
    value: Variable<u32>,
}

#[odra::module]
impl Counter {
    pub fn inc(&mut self) {
        let new_value = self.value.get() + 1;
        self.value.set(new_value);
        emit(Incremented {
            author: env::caller(),
            value: new_value
        });
    }

    pub fn get(&self) -> u32 {
        self.value.get()
    }

    pub fn reset(&mut self) {
        self.value.set(0);
    }
}

#[odra::event]
struct Incremented {
    pub author: Address,
    pub value: u32
}
```

`#[odra::module]` macro derives a `ModuleInstance` implementation that is used to create a `Counter` struct.  

```rust
impl ModuleInstance for Counter {
    fn instance(namespace: &str) -> Self {
        Self {
            value: Variable::new(&format!("{}_{}"), namespace, "value")
        }
    }
}
```

The `Counter` module is now usable.
```rust
let mut counter = Counter::new();
assert_eq!(counter.get(), 0);
counter.inc():
assert_eq!(counter.get(), 1);
assert_eq!(env::events(), vec![Increment { author: env::caller(), value: 1}]);
```

It is possible to use a module more than once. For example, let's define a module that flips a coin and keeps track of the number of heads and tails in two separated `Counter` instances.

```rust
#[odra::module]
struct CoinFlipper {
    heads_count: Counter,
    tails_count: Counter,
}

#[odra::module]
impl CoinFlipper {
    pub fn init(&mut self) {
        self.reset();
    }

    pub fn flip(&mut self) {
        if env::block_time() % 2 == 1 {
            self.heads_count.inc();
        } else {
            self.tails_count.inc();
        }
    }

    pub fn get_stats(&self) -> (u32, u32) {
        (self.heads_count.get(), self.tails_count.get())
    }

    pub fn reset(&self) {
        self.heads_count.set(0);
        self.tails_count.set(0);
    }
}

```
Note that the above module ends up with two named keys defined as `heads_count_value` and `tails_count_value`.

### Contracts
Every module can be used as a building block of another module or as a standalone contract. To utilize a module as a contract, no additional code is required. Every contract automatically derives all the code for contract installation, `no_mangle` functions that a `wasm` file requires, arguments parsing, returning values, and calling the constructor.

Lets define a contract based on the previously defined `CoinFlipper`.

```rust
use flipper::Flipper;

#[odra::module]
struct MyFlipper {
    flipper: Flipper
}

#[odra::module]
impl MyFlipper {

    #[odra::constructor]
    pub fn init(&self) {
        self.flipper.init(10, self.caller());
    }

    pub fn flip(&mut self) {
        self._flip();   
    }
    
    pub fn get_stats(&self) -> (u32, u32) {
        self.flipper.get_stats()
    }

    fn _flip(&mut self) {
        self.flipper.flip()
    }
}

```
All public functions are exposed as contract's entry points, a constructor is annotated with `#[odra::constructor]`, and chooses entry points to expose. Note that the `_filp` method is not exposed as an entry point.

Building the `my_flipper.wasm` file, is as simple as running `cargo build` with project specific feature flags.   

Note that it's effortless to reuse a module that's defined elsewhere.

#### Calling other contracts
To call a contract from another contract a reference to a beforehand deployed item has to be obtained. It can easily be achieved via `at` method.
```rust
let mut coin_flipper = FairCoinFlipper::at(address);
coin_flipper.flip();
```

Yet, that allows for interaction only with a contract written using `odra`. `odra` allows defining custom interfaces for any contract on the Casper platform to overcome that.

```rust
#[odra::external_contract(Token)]
trait TokenInterface {
    fn balance_of(&self, address: Address) -> U256;
}
```

It can be used the same way as it would be a `odra` contract.
```rust
let token = Token::at(token_address);
let my_balance = token.balance_of(env::caller());
```

### Tests
Tests are realized using two in-memory virtual machines. The first one is `CasperVM`, which is intended to test contracts against the same VM that Casper uses. The second VM is `MockVM`, which allows unit-test modules and run the debugger, one of the most community-requested features.

When coding, a developer can use `MockVM` because it's much better for development and debugging, and later validate the code against `CasperVM` without any changes to the code.

All the initialization code for both environments is done under the hood giving developers an easy way of testing multi-contract scenarios. Contract deployment is done via the static `deploy` method, which inherits an argument from the constructor.

Tests are defined in the following way.
```rust
#[odra::test]
fn tests_coin_flipper() {
    let mut flipper = FairCoinFlipper::deploy();
    assert!(heads == 0 && tails == 0);
    flipper.flip()
    let (heads, tails) = flipper.get_stats();
    assert!(heads + tails == 1);
}
```

In addition, the testing framework should allow for interaction with contracts written not using `odra`.

# Odra Standard Library
Along with the framework, we would like to provide a library of reusable modules that can be used out-of-the-box to build large projects. We recognize the following modules as necessary. It is modeled after OpenZeppelin library for Solidity.

- **ERC20** - a module for fungible tokens, that follows ERC20 standard.

- **ERC721** - a module for non-fungible tokens, that follows ERC721 standard.

- **ERC1155** - a module for multi-token, that follows ERC1155 standard. 

- **Ownable** - a module that provides a basic access control mechanism. An account (the owner) can be granted exclusive access to specific functions.

- **Allow List** - a module that allows keeping track of lists of users that are permitted to use a contract.

- **Access Control** - a module that allows implementing role-based access control mechanisms. It is helpful for multi-level permission systems.

- **Pauseable** - a module that allows implementing an emergency stop mechanism that an authorized account can trigger.

- **Ecrecover** - a module that defines the standard implementation of ecrecover. It allows building contracts that use signatures for user authentication.

- **Reentrance Guard** - a module that helps preventing reentrant calls to a function - the most common attack against smart contracts.

- **Wrapped CSPR** - a module that allows to deposit and withdraw CSPR tokens into/from the contract. It implements ERC20 standard.

# Documentation and tutorials
We will provide multiple documents that aim at multiple groups:
- documentation for `odra` core contributors,
- tutorials and examples for smart contract developers,
- tutorials on how to redistribute community-build modules,
- tutorials for Solidity developers that want to switch into Casper.

# Timeline
Proposed list of milestones. Milestones from 3 to 7 can be done in parallel.

## Milestone #1 MVP Part 1
The goal of this milestone is to develop:
- parts: `Variable`, `Mapping`, `Events`,
- macros for modules and contracts: `#[odra::module]`, `#[odra:external_contract]`,
- tests using `CasperVM`,

**Acceptance criteria:**
- version is `0.1.0` released to the http://crates.io,
- it is possible to write simple smart contracts and test them using `CasperVM`.

## Milestone #2 MVP Part 2
The goal of this milestone is to develop:
- `MockVM` to provide debugging features (with breakpoints) and unit-tests not just for whole contracts, but also particular parts and modules.
- a complete example of `ERC20` implementation.

**Acceptance criteria:**
- version is `0.2.0` released to the http://crates.io,
- Rust debugger can be used to debug smart contracts and tests,
- standard library is initialized with the `ERC20` contract and can be used to develop custom tokens.

## Milestone #3 MVP Docs
Milestone #3 aims at:
- choosing a web framework for documentation and tutorials,
- designing the documentation structure,
- complete code documentation for features developed in milestones #1 and #2,
- ERC20-based tutorial.

**Acceptance criteria:**
- `0.1.0` version of documentation and tutorials is published. 

## Milestone #4 CSPR
Milestone #4 is all about supporting CSPR transfers. We will develop the `Wrapped CSPR` Contract and modules for handling CSPR transfers ready to be used via the standard library.

**Acceptance criteria:**
- next version of `odra` is released to the http://crates.io.
- developers have a clear way of handling `CSPR` transfers and holding `CSPR` in smart contract.
- `Wrapped CSPR` is included into standard library.

## Milestone #5 Access Controls
Milestone #5 produces reusable modules for defining access control: 
- Ownable
- Allow list
- Access Control
- Reentrance Guard
- Ecrecover
- Pauseable

**Acceptance criteria:**
- next version of `odra` is released to the http://crates.io.
- above modules are included into standard library.

## Milestone #6 Tokens
Milestone #6 produces reusable modules:
- ERC721
- ERC1155

**Acceptance criteria:**
- next version of `odra` is released to the http://crates.io.
- above modules are included into standard library.

## Milestone #7 Full docs + tutorials
This milestone aims at writing:
- full documentation for standard library modules,
- 0-to-hero tutorial for design patterns for smart contracts, that are modular and reusable,
- tutorial on how to redistribute community-build modules,
- tutorial for Solidity developers that want to switch into Casper.

**Acceptance criteria:**
- next version of documentation and tutorials is published. 

## Milestone #8 Release
At this stage, the code is fully functional and production-ready. To prove that, a full code audit by 3party should be performed and all the aftermath issues resolved.

**Acceptance criteria:**
- version `1.0.0` of `odra` is released to the http://crates.io,
- version `1.0.0` of documentation and tutorials is published.

## Further support
After `1.0.0` is released we intend to use the RFP mechanism to support the framework, which is:
- supporting the community on Discord,
- adjusting `odra` to support Casper 2.0,
- accepting community improvements to the code and documentation.
