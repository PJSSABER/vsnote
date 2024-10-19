# memory layout

calldata
memory
storage

# Scope

public - all can access  (state variable & func)

external - Cannot be accessed internally, only externally, function will only ever be called externally, an interface maybe (func)

internal - only this contract and contracts deriving from it can access (func)

private - can be accessed only from this contract (state variable & func)

# view 
A view function reads state variables but doesn’t write to them.



constants are stored directly in the contract’s bytecode, not in storage, which reduces gas costs for accessing them

Immutable variables are stored directly in the contract’s bytecode

payable is a modifier that can be applied to functions or addresses. When payable is added to a function, that function is able to receive Ether (ETH) during a transaction

msg.value is a special global variable in Solidity. It represents the amount of wei (not Ether, 1 Ether = 10^18 wei) sent in a transaction. It's commonly used inside payable functions to determine how much Ether has been sent

o send ETH while making the function call, we just need to add {} between the name of the function and its parameters  i.e, convert{value: 10}();

Minimum: You must have enough ETH to cover both the transaction fee and the amount you're sending. The smallest unit of ETH is called a "Wei," which is 10^-18 ETH.
Maximum: Limited by the balance in your wallet, minus transaction fees. The largest amount of ETH that can theoretically be held in a single account is 2^256 - 1 wei, an astronomically large number.

When you call a payable function in a smart contract, the ETH is paid from the account (either an Externally Owned Account or another contract) that initiates the call. 

block.number is a global variable in Solidity that indicates the current block's sequential number

A pure function in Solidity is a function that doesn't modify the contract's states, access external contracts, or read the state of the blockchain; it only performs computations on its input parameters and returns a computed result.

To check on the condition, we use the keyword require, then follow by the condition, and a message to report error if the condition is not satisfied.

A state variable in Solidity is a persistent data storage location within a smart contract that maintains its value across multiple function calls and transactions.

Events in Solidity allow you to log and monitor specific changes, such as contract function calls or transactions

When an event is submitted, the event parameters are stored in the transaction log. These logs are associated with the contract’s address and recorded into the blockchain. You can use tools to assist in querying, such as etherscan, etc.Indexed is a keyword used in event parameters in Solidity. It allows these parameters to be logged and searchable, aiding in event filtering. Solidity permits up to three indexed parameters per event.


Only state variables declared as public are accessible from other contracts

use a () to get state variable This is because when you have a public state variable, Solidity actually will create a getter function for you, and when you access the variable from another contract, you’re calling the getter function to retrieve the value.

An enum variable is a value type because its value is stored in uint8, which is a value type

多态： modifier



# error handling

require: Used to validate external inputs and conditions. If a require fails, the remaining gas is refunded to the caller. It's generally used for standard operational checks, like ensuring valid inputs or sufficient balances

revert statement will immediately stop the execution of the current function and undo all changes to the state

assert check statement, If not, it terminates the execution of the function and undoes all changes. Used to handle invariants and internal errors. If an assert fails, all gas is consumed

```solidity
uint val;
try externalContract.f() returns(uint _val) {
    val = _val;
}  catch Error(string memory err) {
  // handle errors caused by require or revert statement
} catch {
  //other conditions
}
```

libraries in Solidity don't have their own state variables. However, they can modify the state of contracts that use them if those contracts pass their state variable as a parameter to the library's functions.

# multi-file

use import to include code from different files

# inheritance
using is
Contract inheritance in Solidity allows a contract to inherit properties and behaviors from another contract. This includes state variables, functions, modifiers, and events, enabling more modular and reusable code.

```solidity
constructor(string name, string symbol) ERC20(name, symbol) { }
```

In Solidity, when contract A inherits from contract B, the code of B is copied into A. To ensure proper initialization, A calls the constructor of B within its own constructor before executing its own constructing steps.

Overriding functions must use the same function name, parameter list, and return type as the function being overridden. Can override only virtual functions in parent
Only those functions that may need to be modified or customized in subcontracts need to be marked as virtual. 

interface is similar to a contract, but it only specifies required functionalities and behaviors without implementation details cannot define variables in interface. like a virtual class.


An abstract contract cannot be instantiated but serves as a base for other contracts. It defines functions, variables, and common functionalities. Abstract contracts can have variables and implementations, while interfaces only have function signatures




In Solidity, when working with structs, we must specify the storage location

# block.timestamp



To let a contract can recieve ether:

- A receive function has no name, takes no parameters, and returns nothing. It must be marked as external and payable
- fallback()

#  ABI
abi.encode VS abi.encodePacked
The main difference is in the data compression.
●
abi.encodePacked is like tightly packing items together, with no extra padding or gaps. This way of packing can save space, but unpacking could be tricky because there are no clear dividers between the items. There even might be multiple ways to unpack.
●
In contrast, abi.encode is like putting items into different bags, and organizing them with standard dividers and padding. Each bag has labels and standards to ensure the integrity of the structure and type of items.