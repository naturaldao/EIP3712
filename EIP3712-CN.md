
---
eip: 3712
title: Standard for Multiple Types of Fungible-Tokens
description: standardize single contracts that support multiple fungible tokens
author: Zhao Li (@1999321), Derek Zhou (@zhous), Yuefei Tan (@whtyfhas)
discussions-to: https://ethereum-magicians.org/t/eip-3712-standard-for-multiple-types-of-fungible-tokens/6912
status: Draft
type: Standards Track
category: ERC
created: 2021-08-01
---

## Abstract（简介）

多重同质化通证标准弥补了[ERC-20](./eip-20.md)和[ERC-1155](./eip-1155.md)的不足之处。在交易多个同质化通证时gas消耗减少，支持多个接收方对多个通证的交易`transferBatchMul`与及多个发送方、多个接收方、多个通证一一对应的交易`transferFromBatchMul`。在授权方面分两种授权：单一同质化通证的数量授权`approve(uint256 id,address spender, uint256 amount)`、全局授权`approve(address spender,bool _status)`。

## Motivation（动机）

ERC20是单一同质化通证标准，当交易涉及多个同质化通证时候，需要加载数量等同的合约。ERC1155把同质性同质化通证和非同质化通证结合在一起，在授权方面缺乏数量方面的授权，对于某一个id的授权与及多个地址同时授权。在交易方面缺乏多个地址对多个通证的转账。本提案弥补ERC20和ERC1155的不足之处，使得其适合多同质化通证进行授权与交易等应用场景。这也就是说本通证标准能够通过一个合约同时管理多个同质化通证，能够让多个发送方对多个接收方的多个通证的交易一次性完成，因此未来dApp数量越多，本标准就越能节省它们所消耗的内存和gas等资源、越能提升合约的综合交互效率——很显然，如果我们能使用一个基于该标准的智能合约为整个行业提供通证发行的通用的无需许可的解决方案，那将给区块链的发展带来很大的启迪，同时对于凸显区块链的效率也将起到非常好的示范作用。

## Specification（提案的接口）

Interfaces: eip.sol
```solidity
pragma solidity ^0.8.0;
pragma abicoder v2;

interface IEIP {
    
    function name(uint256 id) external view returns (string memory);
    
    function symbol(uint256 id) external view returns (string memory);
    
    function decimals(uint256 id) external view returns (uint8);
    
    function totalSupply(uint256 id) external view returns (uint256);

    function balanceOf(uint256 id,address account) external view returns (uint256);

    function transfer(uint256 id,address recipient, uint256 amount) external returns (bool);
    
    function transferBatch(uint256[] memory ids,address recipient,uint256[] memory amounts)external returns(bool);
    
    function transferBatchMul(uint256[] memory ids,address[]memory recipient,uint256[] memory amounts)external returns(bool);

    function allowance(uint256 id,address owner, address spender) external view returns (uint256);

    function approve(uint256 id,address spender, uint256 amount) external returns (bool);
    
    function approve(address spender,bool status) external returns(bool);
    
    function transferFromBatch(uint256[] memory ids,address sender,address recipient,uint256[] memory amounts)external returns(bool);
    
    function transferFromBatchMul(uint256[] memory ids,address[]memory sender,address[]memory recipient,uint256[] memory amounts)external returns(bool);

    function transferFrom(uint256 id,address sender, address recipient, uint256 amount) external returns (bool);

    event Transfer(uint256 id,address indexed from, address indexed to, uint256 value);

    event Approval(uint256 id,address indexed owner, address indexed spender, uint256 value);
    
    event TransferBatch(uint256[] ids,address indexed from,address indexed to,uint256[] value);
    
    event TransferBatchNoIndexed(uint256[] ids,address[] from,address[] to,uint256[] value);
    
    event ApproveBatch(uint256[] ids,address indexed from,address indexed to,uint256[] value);
    
    event ApproveBatchNoIndexed(uint256[] ids,address[] from,address to,uint256[] value);
}
```

`totalSupply`,`balanceOf`,`transfer`,`allowance`,`transferFrom`,`approve(uint256 id,address spender, uint256 amount)`  这几个函数在ERC20的标准基础上增加了第一个参数id来表明同质化通证。

`approve(address spender,bool status)` 纳用ERC1155的全局授权，方便授权于平台方。

`transferBatch` 和 `transferFromBatch` 去掉了ERC1155的 bytes memory data 这个参数，实现捆绑式的转账。

`transferBatchMul` 实现了多个接收方对应多种同质化通证的转账。

`transferFromBatchMul` 实现了多个发送方、多个接收方、多个同质化通证一一对应的授权转账。


## Rationale（基本原理）
 
ERC20用地址来标识每一个同质化通证，ERC1155使用`id`来标识通证，ERC1155相对于ERC20来说，在多个通证转账的过程中节约了gas的费用。本提案使用`id`来标识每一个同质化通证，使得在转账消耗gas方面优于ERC20，因为可以实现打包交易`transferBatch` 或 `transferFromBatch`。由于引入了`id`，且每一个id都在同一个合约里面管理，所以基于同质化通证的性质，增加了数量授权`approve(uint256 id,address spender, uint256 amount)`。对于平台方而言，假定其合约可信，如果通过数量授权，那么需要授权的次数，和需要授权的通证的数量一致，为提高效率故而引入全局状态授权`approve(address spender,bool _status)`，使得平台方可以一次性获得全部授权。同时在一些场景之下，考虑到一个发送方、多个接收方、多个通证的情况与及多个发送方、多个接收方、多个通证的情况分别添加了`transferBatchMul`、`transferFromBatchMul`，扩大本提案可支持的交易类型。

## Backwards Compatibility（向下兼容）

该标准严格按照以太坊发展提案即EIP的要求和技术规范进行扩展，因此我们的标准向下兼容已有的EIP（ERC-1155），即允许旧版通证标准可以在新版通证标准的合约上顺利调用。更具体地说，部分交易功能兼容 ERC-1155 ，但不兼容 ERC-20。

## Reference Implementation（参考实例）

一个签署合约的参考示例：
```solidity

pragma solidity ^0.8.0;
pragma abicoder v2;
import "./eip.sol";

struct ERC20Info{
    uint32 id;
    address manager;
    string name;
    string symbol;
    uint256 totalSupply;
    uint8 decimals;
}

contract ERC20s is IEIP{
    struct ERC20{
        uint32 id;
        address manager;
        string name;
        string symbol;
        uint256 totalSupply;
        uint8 decimals;
        mapping(address => uint256) balances;
        mapping(address => mapping(address => uint256)) allowances;
    }
    mapping(uint256 => ERC20) private allERC20;
    mapping(address => mapping(address => bool)) public approveAll;
    uint32 public nextId = 1;
    
    modifier idInUse(uint256 id) {
        require(bytes(allERC20[id].name).length != 0 || id == 0,"ERC20s:id not in use");
        _;
    }
    // function supportsInterface(bytes4 interfaceId) public view virtual  returns (bool) {
    //     return interfaceId == type(IERC20s).interfaceId
    //         || super.supportsInterface(interfaceId);
    // }
    
    function name(uint256 id) external override view idInUse(id) returns(string memory){
        return allERC20[id].name;
    }
    
    function symbol(uint256 id) external override view idInUse(id) returns(string memory){
        return allERC20[id].symbol;
    }
    
    function decimals(uint256 id) external override view idInUse(id) returns(uint8){
        return allERC20[id].decimals;
    }
    
    function totalSupply(uint256 id) external override view idInUse(id) returns (uint256){
        return allERC20[id].totalSupply;
    }

    function balanceOf(uint256 id,address account) external override view idInUse(id) returns (uint256){
        return allERC20[id].balances[account];
    }

    function transfer(uint256 id,address recipient, uint256 amount) external override idInUse(id) returns (bool){
        _transfer(id,msg.sender,recipient,amount);
        return true;
    }
    
    function transferBatch(uint256[] memory ids,address recipient,uint256[] memory amounts)external override returns(bool){
        _transferBatch(ids,msg.sender,recipient,amounts);
        return true;
    }
    
    function transferBatchMul(uint256[] memory ids,address[]memory recipient,uint256[] memory amounts)external override returns(bool){
        _transferBatchMul(ids,createMsg(msg.sender,ids.length),recipient,amounts);
        return true;
    }

    function allowance(uint256 id,address owner, address spender) external view override returns (uint256){
        return allERC20[id].allowances[owner][spender];
    }

    function approve(uint256 id,address spender, uint256 amount) external override idInUse(id) returns (bool){
        _approve(id,msg.sender,spender,amount);
        return true;
    }
    
    function approve(address spender,bool _status) external override returns(bool){
        approveAll[msg.sender][spender] = _status;
        return true;
    }

    function increaseAllowance(uint256 id,address spender, uint256 addedValue) external idInUse(id) {
        _approve(id,msg.sender,spender,allERC20[id].allowances[msg.sender][spender] + addedValue);
    }
    
    function decreaseAllowance(uint256 id,address spender, uint256 subtractedValue) external idInUse(id) {
        require(allERC20[id].allowances[msg.sender][spender] > subtractedValue,"not enough");
        _approve(id,msg.sender, spender, allERC20[id].allowances[msg.sender][spender] - subtractedValue);
    }
    
    function  transferFrom(uint256 id,address sender, address recipient, uint256 amount) external override idInUse(id) returns(bool) {
        if(approveAll[sender][msg.sender]){
            _transfer(id,sender,recipient,amount);
        }
        else{
            require(allERC20[id].allowances[sender][msg.sender] >= amount,"ERC20:approve not enough");
            _transfer(id,sender,recipient,amount);
            _approve(id,sender,msg.sender,allERC20[id].allowances[sender][msg.sender] - amount);
        }
        return true;
    }
    
    function transferFromBatch(uint256[] memory ids,address sender,address recipient,uint256[] memory amounts)external override returns(bool){
        if(approveAll[sender][msg.sender]){
            _transferBatch(ids,sender,recipient,amounts);
        }
        else{
            for(uint256 i = 0;i < ids.length;i++){
                require(allERC20[ids[i]].allowances[sender][msg.sender] >= amounts[i],"ERC20:approve not enough");
            }
            _transferBatch(ids,sender,recipient,amounts);
            _approveBatch(ids,sender,msg.sender,amounts);
        }
        return true;
    }
    
    function transferFromBatchMul(uint256[] memory ids,address[]memory sender,address[]memory recipient,uint256[] memory amounts)external override returns(bool){
        bool _status = true;
        for(uint256 i = 0;i < ids.length;i++){
            if(!approveAll[sender[i]][msg.sender])
                _status = false;
        }
        if(_status){
            _transferBatchMul(ids,sender,recipient,amounts);
        }
        else{
            for(uint256 i = 0;i < ids.length;i++){
                require(allERC20[ids[i]].allowances[sender[i]][msg.sender] >= amounts[i],"ERC20:approve not enough");
            }
            _transferBatchMul(ids,sender,recipient,amounts);
            _approveBatch(ids,sender,msg.sender,amounts);
        }
        return true;
    }
    
    function _transfer(uint256 id,address sender, address recipient, uint256 amount) internal virtual {
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");
        
        allERC20[id].balances[sender] = allERC20[id].balances[sender] - amount;
        allERC20[id].balances[recipient] = allERC20[id].balances[recipient] + amount;
        emit Transfer(id,sender, recipient, amount);
    }
    
    function _transferBatch(uint256[] memory ids,address sender,address recipient,uint256[] memory amounts) internal virtual{
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");
        
        require(ids.length == amounts.length,"ERC20s: length not equal");
        for(uint256 i = 0;i < ids.length;i++){
            if(amounts[i] == 0){
                continue;
            }
            require(bytes(allERC20[ids[i]].name).length != 0 || ids[i] == 0,"ERC20s: ERC20 not in use");
            allERC20[ids[i]].balances[sender] = allERC20[ids[i]].balances[sender] - amounts[i];
            allERC20[ids[i]].balances[recipient] = allERC20[ids[i]].balances[recipient] + amounts[i];
        }
        emit TransferBatch(ids,sender,recipient,amounts);
    }
    
    function _transferBatchMul(uint256[] memory ids,address[]memory sender,address[]memory recipient,uint256[] memory amounts) internal virtual{
        
        require(ids.length == amounts.length && sender.length == recipient.length && sender.length == amounts.length,"ERC20s: length not equal");
        for(uint256 i = 0;i < ids.length;i++){
            require(sender[i] != address(0), "ERC20: transfer from the zero address");
            require(recipient[i] != address(0), "ERC20: transfer to the zero address");
            require(bytes(allERC20[ids[i]].name).length != 0 || ids[i] == 0,"ERC20s: ERC20 not in use");
            if(amounts[i] == 0){
                continue;
            }
            allERC20[ids[i]].balances[sender[i]] = allERC20[ids[i]].balances[sender[i]] - amounts[i];
            allERC20[ids[i]].balances[recipient[i]] = allERC20[ids[i]].balances[recipient[i]] + amounts[i];
        }
        emit TransferBatchNoIndexed(ids,sender,recipient,amounts);
    }
    
    function _approve(uint256 id,address owner, address spender, uint256 amount) internal virtual {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");

        allERC20[id].allowances[owner][spender] = amount;
        emit Approval(id,owner, spender, amount);
    }
    
    function _approveBatch(uint256[] memory ids,address owner,address spender,uint256[] memory amounts)internal virtual{
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");
        
        require(ids.length == amounts.length,"ERC20s: length not equal");
        for(uint256 i = 0;i < ids.length;i++){
            require(bytes(allERC20[ids[i]].name).length != 0 || ids[i] == 0,"ERC20s: ERC20 not in use");
            if(amounts[i] == 0){
                continue;
            }
            allERC20[ids[i]].allowances[owner][spender] = allERC20[ids[i]].allowances[owner][spender] - amounts[i];
            amounts[i] = allERC20[ids[i]].allowances[owner][spender];
        }
        
        emit ApproveBatch(ids,owner,spender,amounts);
    }
    
    function _approveBatch(uint256[] memory ids,address[] memory owner,address spender,uint256[] memory amounts)internal virtual{
        require(ids.length == amounts.length  && owner.length == amounts.length,"ERC20s: length not equal");
        
        for(uint256 i = 0;i < ids.length;i++){
            require(bytes(allERC20[ids[i]].name).length != 0 || ids[i] == 0,"ERC20s: ERC20 not in use");
            if(amounts[i] == 0){
                continue;
            }
            allERC20[ids[i]].allowances[owner[i]][spender] = allERC20[ids[i]].allowances[owner[i]][spender] - amounts[i];
            amounts[i] = allERC20[ids[i]].allowances[owner[i]][spender];
        }
        
        emit ApproveBatchNoIndexed(ids,owner,spender,amounts);
    }
    
    function createMsg(address _msg,uint256 len) internal view returns(address[] memory data){
        data = new address[](len);
        for(uint256 i = 0;i < len;i++){
            data[i] = _msg;
        }
    }
    
    function _createERC20(uint256 id,string memory name,string memory symbol,uint8 _decimals) internal {
        require(bytes(allERC20[id].name).length == 0,"ERC20s: ERC20 in use");
        allERC20[id].name = name;
        allERC20[id].symbol = symbol;
        allERC20[id].decimals = _decimals;
    }
}
```

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
 
