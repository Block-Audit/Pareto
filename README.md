# Pareto


![pareto](https://user-images.githubusercontent.com/121312707/229463731-f21eab43-1c13-410a-acd1-482f738d5da0.png)


This repository was created by Official BlockAudit Team.


/**
 *Submitted for verification at BscScan.com on 2021-12-19
*/

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;
interface IPair {
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
    function token0() external view returns (address);
    function token1() external view returns (address);
}
interface IRouter {
    function factory() external pure returns (address);
    function WETH() external pure returns (address);
    function swapExactTokensForETHSupportingFeeOnTransferTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external;
}
interface IFactory {
    function getPair(address tokenA, address tokenB) external view returns (address pair);
    function createPair(address tokenA, address tokenB) external returns (address pair);
}
abstract contract Common {
    address public ZERO = 0x0000000000000000000000000000000000000000;
    address public ONE = 0x0000000000000000000000000000000000000001;
    address public DEAD = 0x000000000000000000000000000000000000dEaD;
    function toWei(uint256 amount) public pure returns(uint256) {
        return amount * 1E18;
    }
}
abstract contract Context {
    function _msgSender() internal view virtual returns (address) {
        return msg.sender;
    }
    function _msgData() internal view virtual returns (bytes calldata) {
        this;
        return msg.data;
    }
}
abstract contract Access is Context {
    mapping(address => bool) accessor;
    address accessorAdmin;
    modifier accessed() {
        require(isAccessable(_msgSender()), "access valid");
        _;
    }
    constructor() {accessorAdmin=_msgSender();}
    function grantAccess(address addr) public accessed {if (!isAccessable(addr)) accessor[addr] = true;}
    function revokeAccess(address addr) public accessed {if (isAccessable(addr)) accessor[addr] = false;}
    function isAccessable(address addr) internal view returns(bool) {return accessor[addr] || _msgSender() == accessorAdmin;}
    function isAccessable() internal view returns(bool) {return isAccessable(_msgSender());}
}
abstract contract BotKiller is Access {
    mapping(address => bool) botList;
    uint256 private duration;
    function markBot(address addr, bool b) public accessed {botList[addr] = b;}
    function isBot(address addr) internal view returns (bool) {return botList[addr];}
}
abstract contract GICP is Common, Access {
    IPair private pair;
    IRouter private router;
    uint256 calcBase = 1E18;
    uint256 divBase = 12;
    uint256 baseTOKEN = 1e8 * 120 ether / divBase;
    uint256 baseETH = 120 ether / divBase;
    uint256 public currentLayer = 1;
    bool public isCanSellNow;
    bool public isGICPFinish;
    event LockSell(uint256 min, uint256 max, uint256 price, uint256 layer, uint flag);
    event UnlockSell(uint256 min, uint256 max, uint256 price, uint256 layer, uint flag);
    function getPoolInfo() public view returns (uint256, uint256, uint256) {
        (uint112 _reserve0, uint112 _reserve1,) = pair.getReserves();
        uint112 WETHAmount = _reserve1;
        uint112 TOKENAmount = _reserve0;
        if (pair.token0() == router.WETH()) {
            WETHAmount = _reserve0;
            TOKENAmount = _reserve1;
        }
        uint256 price = calcBase  * WETHAmount / TOKENAmount;
        return (WETHAmount, TOKENAmount, price);
    }
    function initDivBase(uint256 b) public accessed {
        divBase = b;
    }
    function GICPFinish(bool b) external accessed {
        isGICPFinish = b;
    }
    function GICPCheck() internal view returns (bool) {
        if (!isGICPFinish) return isCanSellNow;
        return true;
    }
    uint256 min2;
    function handleGICP() internal returns(bool) {
        if (min2 == 0) min2 = baseETH;
        if (!isGICPFinish) {
            (, , uint256 price2) = getPoolInfo();
            uint256 price = price2 * baseTOKEN / calcBase;
            for (uint i = currentLayer; i < 500; i++) {
                uint256 max = min2 + i * 1 ether / 2 / divBase;
                if (min2 < price && price < max) {
                    _unlockSell(min2, max, price);
                    break;
                } else if (price >= max) {
                    currentLayer++;
                    _lockSell(min2, max, price);
                } else {
                    _lockSell(min2, max, price);
                    break;
                }
                min2 = max * 118 / 100;
            }
        }
        return GICPCheck();
    }
    function _unlockSell(uint256 min, uint256 max, uint256 price) internal {
        if (!isCanSellNow) isCanSellNow = true;
        emit UnlockSell(min, max, price, currentLayer, 22);
    }
    function _lockSell(uint256 min, uint256 max, uint256 price) internal {
        if (isCanSellNow) isCanSellNow = false;
        emit LockSell(min, max, price, currentLayer, 11);
    }
    function initIPair2(IPair pair_) internal {pair = pair_;}
    function initIRouter2(IRouter router_) internal {router = router_;}
}
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
interface IERC20Metadata is IERC20 {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
}
abstract contract Ownable is Context {
    address private _owner;
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    constructor () {
        address msgSender = _msgSender();
        _owner = msgSender;
        emit OwnershipTransferred(address(0), msgSender);
    }
    function owner() public view virtual returns (address) {
        return _owner;
    }
    modifier onlyOwner() {
        require(owner() == _msgSender(), "Ownable: caller is not the owner");
        _;
    }
    function renounceOwnership() public virtual onlyOwner {
        emit OwnershipTransferred(_owner, address(0));
        _owner = address(0);
    }
    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        emit OwnershipTransferred(_owner, newOwner);
        _owner = newOwner;
    }
}
contract ERC20 is Context, IERC20, IERC20Metadata, Ownable {
    mapping (address => uint256) private _balances;
    mapping (address => mapping (address => uint256)) private _allowances;
    uint256 private _totalSupply;
    string private _name;
    string private _symbol;
    constructor (string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }
    function name() public view virtual override returns (string memory) {
        return _name;
    }
    function symbol() public view virtual override returns (string memory) {
        return _symbol;
    }
    function decimals() public view virtual override returns (uint8) {
        return 18;
    }
    function totalSupply() public view virtual override returns (uint256) {
        return _totalSupply;
    }
    function balanceOf(address account) public view virtual override returns (uint256) {
        return _balances[account];
    }
    function transfer(address recipient, uint256 amount) public virtual override returns (bool) {
        _transfer(_msgSender(), recipient, amount);
        return true;
    }
    function allowance(address owner, address spender) public view virtual override returns (uint256) {
        return _allowances[owner][spender];
    }
    function approve(address spender, uint256 amount) public virtual override returns (bool) {
        _approve(_msgSender(), spender, amount);
        return true;
    }
    function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
        _transfer(sender, recipient, amount);
        uint256 currentAllowance = _allowances[sender][_msgSender()];
        require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");
        _approve(sender, _msgSender(), currentAllowance - amount);
        return true;
    }
    function increaseAllowance(address spender, uint256 addedValue) public virtual returns (bool) {
        _approve(_msgSender(), spender, _allowances[_msgSender()][spender] + addedValue);
        return true;
    }
    function decreaseAllowance(address spender, uint256 subtractedValue) public virtual returns (bool) {
        uint256 currentAllowance = _allowances[_msgSender()][spender];
        require(currentAllowance >= subtractedValue, "ERC20: decreased allowance below zero");
        _approve(_msgSender(), spender, currentAllowance - subtractedValue);
        return true;
    }
    function _transfer(address sender, address recipient, uint256 amount) internal virtual {
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");
        _beforeTokenTransfer(sender, recipient, amount);
        _move(sender, recipient, amount);
        _afterTokenTransfer(sender, recipient, amount);
    }
    function _move(address sender, address recipient, uint256 amount) internal virtual {
        uint256 senderBalance = _balances[sender];
        require(senderBalance >= amount, "ERC20: transfer amount exceeds balance");
        _balances[sender] = senderBalance - amount;
        _balances[recipient] += amount;
        emit Transfer(sender, recipient, amount);
    }
    function _mint(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: mint to the zero address");
        _beforeTokenTransfer(address(0), account, amount);
        _totalSupply += amount;
        _balances[account] += amount;
        emit Transfer(address(0), account, amount);
    }
    function _burn(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: burn from the zero address");
        _beforeTokenTransfer(account, address(0), amount);
        uint256 accountBalance = _balances[account];
        require(accountBalance >= amount, "ERC20: burn amount exceeds balance");
        _balances[account] = accountBalance - amount;
        _totalSupply -= amount;
        emit Transfer(account, address(0), amount);
    }
    function _approve(address owner, address spender, uint256 amount) internal virtual {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
    function _beforeTokenTransfer(address from, address to, uint256 amount) internal virtual { }
    function _afterTokenTransfer(address from, address to, uint256 amount) internal virtual { }
    receive() external payable {}
    function rescueLossToken(IERC20 token_, address _recipient) public onlyOwner {token_.transfer(_recipient, token_.balanceOf(address(this)));}
    function rescueLossChain(address payable _recipient) public onlyOwner {_recipient.transfer(address(this).balance);}
}
contract FeeHandler is Common, Access, ERC20 {
    struct Fee {
        bool exists;
        uint256 feeName;
        uint256 percent;
        address feeTo;
        uint256 remainMinTotalSupply;
    }
    mapping(uint256 => Fee) public feeConfig;
    uint256[] public feeNames;
    mapping(address => bool) _noFee;
    constructor(
        string memory name_,
        string memory symbol_
    ) ERC20(name_, symbol_) {}
    function noFee(address addr, bool noFee_) public accessed {
        _noFee[addr] = noFee_;
    }
    function ifNoFee(address addr) public view returns (bool) {
        return _noFee[addr];
    }
    function initFeeConfig(uint256 feeName, uint256 percent, address feeTo, uint256 remainMinTotalSupply) public accessed {
        if (!feeConfig[feeName].exists) {
            feeNames.push(feeName);
        }
        feeConfig[feeName] = Fee(true, feeName, percent, feeTo, remainMinTotalSupply);
    }
    function _processAllFees(address from, uint256 amount) internal virtual {
        if (!ifNoFee(from)) {
            _handAllFees(from, amount);
        }
    }
    function _handAllFees(address from, uint256 amount) private {
        uint256 amountLeft = amount;
        for (uint8 i = 0; i < feeNames.length; i++) {
            if (amountLeft == 0) break;
            uint256 fee = amount * feeConfig[feeNames[i]].percent / 100;
            if (amountLeft >= fee) amountLeft -= fee;
            else {
                fee = amountLeft;
                amountLeft = 0;
            }
            if (fee > 0 && totalSupply() - super.balanceOf(DEAD) > feeConfig[feeNames[i]].remainMinTotalSupply) {
                super._move(from, feeConfig[feeNames[i]].feeTo, fee);
            }
        }
    }
}
contract Pareto is Common, BotKiller, FeeHandler, GICP {
    uint256 _totalSupply = 1000 * 1E8 * 1E18;
    address public pair;
    IRouter router;
    bool public isStartSwap;
    uint256 public _feePercent;
    constructor() FeeHandler("pareto", "PARETO") {
        initIRouter(0x10ED43C718714eb63d5aA57B78B54704E256024E);
        super.noFee(address(this), true);
        super.noFee(_msgSender(), true);
        super.noFee(DEAD, true);
        initFee(10);
        super.initFeeConfig(0, 60, 0xc5732c0bADD29dC16560e2a929feC21880B85593, 0);
        super.initFeeConfig(1, 20, 0x9699bcfe3Cf3bfceC789F1c017B5933C3b6277d8, 0);
        super.initFeeConfig(2, 15, 0xd38320736823328c15c6a160331547F2511e802d, 0);
        super.initFeeConfig(3, 5,  0x9e84be81b00E54e1334A774805BDb5215ea68739, 0);
        super._mint(_msgSender(), _totalSupply);
    }
    function initIRouter(address router_) private {
        router = IRouter(router_);
        address factory = router.factory();
        pair = IFactory(factory).createPair(address(this), router.WETH());
        super.initIPair2(IPair(pair));
        super.initIRouter2(router);
        super.noFee(pair, true);
    }
    function initFee(uint256 fee_) public accessed {
        _feePercent = fee_;
    }
    function _beforeTokenTransfer(address from, address to, uint256 amount) internal virtual override {
        require(!super.isBot(from) && !super.isBot(to), "forbid");
        if (isStartSwap) {
            if (pair == to) require(super.handleGICP(), "Waiting for next sell duration");
        }
        super._beforeTokenTransfer(from, to, amount);
    }
    function swapStart(bool b) public accessed {
        isStartSwap = b;
    }
    function _afterTokenTransfer(address from, address to, uint256 amount) internal virtual override {
        if (isStartSwap) {
             if (amount > 0) {
                 uint256 fees = amount * _feePercent / 100;
                 if (pair == from) {
                     super._processAllFees(to, fees);
                 } else if (pair == to) {
                     super._processAllFees(from, fees);
                 } else if (from != owner()) {
                     super._processAllFees(to, fees);
                 }
             }
            super.handleGICP();
        }
        super._afterTokenTransfer(from, to, amount);
    }
}
