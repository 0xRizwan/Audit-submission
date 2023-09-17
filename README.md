## Title
In `ContractFactory.sol`, `deploy()` may not work on `zksync Era` with current implementation

## Vulnerability Detail

Allo V2 is intended to deployed on All EVM compatible chains + Zkync Era.  This report discuss an issue with zksync era with regards to current implementation of `deploy()` in `ContractFactory.sol`

```Solidity
File: allo-v2/contracts/factory/ContractFactory.sol

    function deploy(string memory _contractName, string memory _version, bytes memory creationCode)
        external
        payable
        onlyDeployer
        returns (address deployedContract)
    {
        // hash salt with the contract name and version
        bytes32 salt = keccak256(abi.encodePacked(_contractName, _version));

        // ensure salt has not been used
        if (usedSalts[salt]) revert SALT_USED();

        usedSalts[salt] = true;

        deployedContract = CREATE3.deploy(salt, creationCode, msg.value);

        emit Deployed(deployedContract, salt);
    }
```

According to ZKSync Era's documentation, the bytecode of a contract must be known beforehand by the compiler for the CREATE/CREATE2 opcode to work:

> To guarantee that create/create2 functions operate correctly, the compiler must be aware of the bytecode of the deployed contract in advance. The compiler interprets the calldata arguments as incomplete input for ContractDeployer, as the remaining part is filled in by the compiler internally.

However, in `deploy()` function of `ContractFactory.sol` contract, the contract's bytecode is provided by the caller as an argument. It can be seen the bytes code,

```Solidity
    function deploy(string memory _contractName, string memory _version, bytes memory creationCode)
```

passed as argument in 

```Solidity
        deployedContract = CREATE3.deploy(salt, creationCode, msg.value);
```

`CREATE3` from solady uses `CREATE2` which can be checked [here](https://github.com/Vectorized/solady/blob/77809c18e010b914dde9518956a4ae7cb507d383/src/utils/CREATE3.sol#L63-L118)

```Solidity
File: src/utils/CREATE3.sol

63    function deploy(bytes32 salt, bytes memory creationCode, uint256 value)
        internal
        returns (address deployed)
    {


       // some code


            let proxy := create2(0, 0x10, 0x10, salt)


         // some code


            if iszero(
                call(
                    gas(), // Gas remaining.
                    proxy, // Proxy's address.
                    value, // Ether value.
98                  add(creationCode, 0x20), // Start of `creationCode`.
99                  mload(creationCode), // Length of `creationCode`.
                    0x00, // Offset of output.
                    0x00 // Length of output.
                )
            ) {


          // some code


118    }
```

Therefore, deploy() function will not work on ZKSync due to the following official reason by zkSync.

![zksync](https://github.com/sherlock-audit/2023-09-Gitcoin-mohammedrizwann123/assets/112799398/74b205b9-bf76-4616-9a26-4f06687625ae)

## Recommendation
Refer the zkSync [documentation](https://era.zksync.io/docs/reference/architecture/differences-with-ethereum.html#create-create2) to to comply the `create` design consideration for zkSync as explained above.
