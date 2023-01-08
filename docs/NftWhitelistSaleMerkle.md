# NftWhitelistSaleMerkle

## Contract Prologue

``` ts linenums="1"
contract NftWhitelistSaleMerkle is Sample {
    ///////////////////////////////
    //    PRESALE CODE STARTS    //
    ///////////////////////////////

    uint256 public presaleActiveTime = 0;
    uint256 public presaleMaxMint = 2;
    uint256 public presaleAddressLimit = 2;
    bytes32 public whitelistMerkleRoot;
    mapping(address => uint256) public presaleClaimedBy;
```

`L6 [QA/Declarations/Low]` – Integer slots are initialized to `0` by default. The proper declaration would be 

```uint256 public presaleActiveTime;```

`L6 [Logic/Medium]` – Setting presale start time to `0`, in conjuction with logic contained in `purchaseTokensPresale` (`block.timestamp` compared against `0`, which is always `true`), activates the presale immediately after deployment. Unless this is the intent, this is a potential violation of the business logic of the contract. 

## purchaseTokensPresale
``` ts linenums="1"
function purchaseTokensPresale(uint256 _howMany, bytes32[] calldata _proof) external payable {
    uint256 supply = totalSupply();
    require(supply + _howMany + adminMint <= maxSupply, "Max NFT limit exceeded");

    require(inWhitelist(_proof, msg.sender), "You are not in presale");
    require(block.timestamp > presaleActiveTime, "Presale is not active");

    presaleClaimedBy[msg.sender] += _howMany;

    require(presaleClaimedBy[msg.sender] <= presaleMaxMint, "Max mint amount per session exceeded");
    uint256 ownerMintedCount = addressMintedBalance[msg.sender];
    require(ownerMintedCount + _howMany <= presaleAddressLimit, "max NFT per address exceeded");

    for (uint256 i = 1; i <= _howMany; i++) {
        addressMintedBalance[msg.sender]++;
        _safeMint(msg.sender, 1);
    }
}
```

`L2 [QA/Gas/Low]` – Consider removing the `supply` variable and calling `totalSupply()` directly in the code of the following `require` statement. Since `supply` is only used once in the function body, there's no need to instantiate a variable.

`L10 [QA/Strings/Low]` – Consider shortening the error message to 32 characters or below to optimize deployment and run-time footprints.

`L11 [QA/Consistency/Low]` – Consider capitalizing "max" for consistency. 

`L14-L17 [QA/Gas/High]` – The actual minting loop violates the idea behind ERC721A: to optimize gas costs when batch minting tokens. Every iteration increments a storage variable and mints a "batch of 1", thus forcing the minters to pay excessive gas. The proper implementation (no looping):

``` ts
    addressMintedBalance[msg.sender] += _howMany;
    _safeMint(msg.sender, _howMany);
```

The very same issue exists in the `purchaseTokens` function [Sample](Sample.md).