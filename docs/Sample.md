# Sample

## Contract Prologue

``` ts linenums="1"
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.15;

import "erc721a@3.3.0/contracts/ERC721A.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/common/ERC2981.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

interface OpenSea {
    function proxies(address) external view returns (address);
}

contract Sample is ERC721A("Sample", "SampleNFT"), Ownable, ERC2981 {
    using Strings for uint256;

    bool public revealed = false;
    string public notRevealedMetadataFolderIpfsLink;
    uint256 public maxMintAmount = 1;
    uint256 public maxSupply = 3900;
    uint256 public adminMint = 50;
    string public metadataFolderIpfsLink;
    uint256 public addressLimit = 1;
    mapping(address => uint256) public addressMintedBalance;
    string constant baseExtension = ".json";
    uint256 public publicmintActiveTime = 0;
```

`L17 [QA/Declarations/Low]` – Boolean slots are initialized to `false` by default. The proper declaration would be 

```bool public revealed;``` 

`L26 [QA/Declarations/Low]` – Integer slots are initialized to `0` by default. The proper declaration would be 

```uint256 public publicmintActiveTime;```

`L26 [QA/Declarations/Low]` – Slot name doesn't follow the camelCase naming scheme that other slot names follow here. Suggested change:

```uint256 public publicMintActiveTime;```

## purchaseTokens

``` ts linenums="1"    
function purchaseTokens(uint256 _mintAmount) public payable {
    require(block.timestamp > publicmintActiveTime, "The contract is paused");
    uint256 supply = totalSupply();
    require(_mintAmount > 0, "You have to mint at least 1 NFT");
    require(_mintAmount <= maxMintAmount, "Max mint amount per session exceeded");
    uint256 ownerMintedCount = addressMintedBalance[msg.sender];
    require(ownerMintedCount + _mintAmount <= addressLimit, "max NFT per address exceeded");
    require(supply + _mintAmount + adminMint <= maxSupply, "Max NFT limit exceeded");

    for (uint256 i = 1; i <= _mintAmount; i++) {
        addressMintedBalance[msg.sender]++;
        _safeMint(msg.sender, 1);
    }
}
```

`L3 [QA/Gas/Low]` – However small of a gas footprint instantiating a memory variable to store the value of `totalSupply()` is, it's non-zero, and in this case the variable is instantiated prematurely, 5 statements before it's actually needed, of which 3 statements might revert. Recommended change: declare `supply` right before use or simply call `totalSupply()` directly in the corresponding statement (`L8`) since this value is only used once in `purchaseToken`.

`L5 [QA/Strings/Low]` – Consider shortening the error message to 32 characters or below to optimize deployment and run-time footprints.

`L7 [QA/Consistency/Low]` – Consider capitalizing "max" for consistency. 

`L10-L13 [QA/Gas/High]` – The actual minting loop violates the idea behind ERC721A: to optimize gas costs when batch minting tokens. Every iteration increments a storage variable and mints a "batch of 1", thus forcing the minters to pay excessive gas. The proper implementation (no looping):

``` ts
    addressMintedBalance[msg.sender] += _mintAmount;
    _safeMint(msg.sender, _mintAmount);
```

The very same issue exists in the `purchaseTokensPresale` function [NftWhitelistSaleMerkle](NftWhitelistSaleMerkle.md)

## Mint

``` ts
function Mint(address[] calldata _sendNftsTo, uint256 _howMany) external onlyOwner {
    adminMint -= _sendNftsTo.length * _howMany;

    for (uint256 i = 0; i < _sendNftsTo.length; i++) _safeMint(_sendNftsTo[i], _howMany);
}
```

`L1 [QA/Declarations/Low]` – Function name doesn't follow the camelCase naming scheme. Suggested change: `mint`.

## setadminMint

``` ts
function setadminMint(uint256 _newadminMint) public onlyOwner {
    adminMint = _newadminMint;
}
```

`L1 [QA/Declarations/Low]` - Function name doesn't follow the camelCase naming scheme. Suggested change: `setadminMint`.

`L1-L2 [QA/Declarations/Low]` - Variable name doesn't follow the camelCase naming scheme. Suggested change: `_newAdminMint`.

## revealFlip
``` ts
function revealFlip() public onlyOwner {
    revealed = !revealed;
}
```

`[QA/Best Practices/Medium]` The reveal function should be once-only. Concealing the art after reveal might unnecessarily signal shady intentions to the community. Suggested change:

``` ts
function reveal() public onlyOwner {
    revealed = true;
}
```

## setAddressLimit
``` ts linenums="1"
function setaddressLimit(uint256 _limit) public onlyOwner {
    addressLimit = _limit;
}
```

Function name doesn't follow the camelCase naming scheme. Suggested change: `setAddressLimit`.
