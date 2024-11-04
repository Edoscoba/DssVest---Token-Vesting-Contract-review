| Client         | MakerDAO                                        |
| :------------- | :---------------------------------------------- |
| Title          | Smart Contract Audit Report                     |
| Target         | DssVest - Token Vesting Contract                |
| Author         | [Obi Simon](https://github.com/Edoscoba)        |
| Classification | Public                                          |
| Status         | Draft                                           |
| Date Created   | November 2, 2024                                |


## DssVest - Token Vesting Contract
Token Vesting Contract is a token vesting system designed to support a range of vesting schedules, typically used to distribute tokens over time to employees, early investors, or contributors. Vesting is a common practice in blockchain and startups where tokens or shares are gradually unlocked based on time or performance milestones, ensuring commitment and alignment of interests.

### Overview
The vesting system is composed of the base contract (DssVest) and three derived contract which are:

- DssVestMintable: A version that mints tokens directly to users as they vest.

- DssVestSuckable: A version specifically for DAI (the stablecoin used in MakerDAO), which interacts with MakerDAO’s core system to generate and transfer DAI to users.

- DssVestTransferrable: A version for pre-minted tokens that transfers tokens from the contract’s balance to the user.

- The base DssVest contract provides essential functionality, including creating and managing vesting awards, checking vesting progress, and handling restricted vesting scenarios.
## CONTRACT REVIEW 

This contract contains the following Interfaces:
```solidity
interface MintLike {
    function mint(address, uint256) external;
}
```
This line Defines a MintLike interface with a mint function, used to interact with a token’s minting function.

```solidity
interface ChainlogLike {
    function getAddress(bytes32) external view returns (address);
}
```
This line defines a ChainlogLike interface for fetching addresses of key components, used in DssVestSuckable.

```solidity
interface DaiJoinLike {
    function exit(address, uint256) external;
}
```
Defines DaiJoinLike for Dai token transfer and withdrawal functionality.

```solidity
interface VatLike {
    function hope(address) external;
    function suck(address, address, uint256) external;
    function live() external view returns (uint256);
}
```
Defines VatLike for core MakerDAO functions like suck (minting DAI) and hope (authorizing addresses for DAI transfers).

```solidity
interface TokenLike {
    function transferFrom(address, address, uint256) external returns (bool);
}
```
This line defines TokenLike to interact with ERC20 tokens using transferFrom.

### DssVest Contract Body
The DssVest abstract contract defines the base structure of vesting data and essential authorization mechanisms.

#### Data Structures and State Variables:

```solidity
mapping (address => uint256) public wards;
```
This mapping stores authorized addresses for modifying the contract.

```solidity
struct Award {
    address usr;   // Vesting recipient
    uint48  bgn;   // Vesting start timestamp
    uint48  clf;   // Cliff timestamp (earliest claim date)
    uint48  fin;   // Vesting end timestamp
    address mgr;   // Manager authorized to revoke vesting
    uint8   res;   // Restriction flag
    uint128 tot;   // Total reward amount
    uint128 rxd;   // Amount claimed so far
}
```
This line defines the Award struct, which contains vesting details for each award.

```solidity
mapping (uint256 => Award) public awards;
```
Maps vesting contract IDs to their corresponding Award struct.

```solidity
uint256 public cap;
```
This public variable sets a maximum cap for token issuance rate.

```solidity
uint256 public ids;

```
Counts total number of vesting contracts created.

```solidity
uint256 internal locked;
```
A mutex lock used to prevent re-entrancy in critical functions.

```solidity
uint256 public constant  TWENTY_YEARS = 20 * 365 days;
```
This public constant defines TWENTY_YEARS as a time constraint constant used throughout vesting setup functions.

#### Events
```solidity
event Rely(address indexed usr);
event Deny(address indexed usr);
event File(bytes32 indexed what, uint256 data);
event Init(uint256 indexed id, address indexed usr);
event Vest(uint256 indexed id, uint256 amt);
event Restrict(uint256 indexed id);
event Unrestrict(uint256 indexed id);
event Yank(uint256 indexed id, uint256 end);
event Move(uint256 indexed id, address indexed dst);
```
The event is just used to emit signals for blockchain observers, making it easier to track significant contract operations.

### Function Definitions
#### Getter functions

```solidity
function usr(uint256 _id) external view returns (address) { return awards[_id].usr; }
function bgn(uint256 _id) external view returns (uint256) { return awards[_id].bgn; }
function clf(uint256 _id) external view returns (uint256) { return awards[_id].clf; }
function fin(uint256 _id) external view returns (uint256) { return awards[_id].fin; }
function mgr(uint256 _id) external view returns (address) { return awards[_id].mgr; }
function res(uint256 _id) external view returns (uint256) { return awards[_id].res; }
function tot(uint256 _id) external view returns (uint256) { return awards[_id].tot; }
function rxd(uint256 _id) external view returns (uint256) { return awards[_id].rxd; }
```
These are getter function for each of the primary fields within the Award struct, returning specific attributes for a given vesting contract.

#### Constructor
```solidity
constructor() public {
    wards[msg.sender] = 1;
    emit Rely(msg.sender);
}
```
The constructor initializes the contract, authorizing the contract deployer by setting their wards value to 1.

#### Modifiers
```solidity
modifier lock {
    require(locked == 0, "DssVest/system-locked");
    locked = 1;
    _;
    locked = 0;
}
```
This modifier function prevents re-entrancy by setting a lock before executing sensitive functions.

```solidity
modifier auth {
    require(wards[msg.sender] == 1, "DssVest/not-authorized");
    _;
}
```
Restricts access to authorized users by checking wards mapping.

#### Authorization Functions
```solidity
function rely(address _usr) external auth {
    wards[_usr] = 1;
    emit Rely(_usr);
}
```
This function grants authorization to a specified address.

```solidity
function deny(address _usr) external auth {
    wards[_usr] = 0;
    emit Deny(_usr);
}
```
This function Revokes authorization from a specified address.

####Configuration Functions
```solidity
function file(bytes32 what, uint256 data) external lock auth {
    if (what == "cap") cap = data;
    else revert("DssVest/file-unrecognized-param");
    emit File(what, data);
}
```
This function allows authorized users to update the contract’s cap parameter, enforcing a maximum token issuance rate.

#### Math Utility Functions
```solidity
function min(uint256 x, uint256 y) internal pure returns (uint256 z) {
    z = x > y ? y : x;
}
function add(uint256 x, uint256 y) internal pure returns (uint256 z) {
    require((z = x + y) >= x, "DssVest/add-overflow");
}
function sub(uint256 x, uint256 y) internal pure returns (uint256 z) {
    require((z = x - y) <= x, "DssVest/sub-underflow");
}
function mul(uint256 x, uint256 y) internal pure returns (uint256 z) {
    require(y == 0 || (z = x * y) / y == x, "DssVest/mul-overflow");
}
```
These are  various math functions (min, add, sub, and mul) with overflow checks to prevent erroneous values.

#### Award Creation
```solidity
function create(address _usr, uint256 _tot, uint256 _bgn, uint256 _tau, uint256 _eta, address _mgr) external lock auth returns (uint256 id) {
    require(_usr != address(0), "DssVest/invalid-user");
    require(_tot > 0, "DssVest/no-vest-total-amount");
    require(_bgn < add(block.timestamp, TWENTY_YEARS), "DssVest/bgn-too-far");
    require(_bgn > sub(block.timestamp, TWENTY_YEARS), "DssVest/bgn-too-long-ago");
    require(_tau > 0, "DssVest/tau-zero");
    require(_tot / _tau <= cap, "DssVest/rate-too-high");
    require(_tau <= TWENTY_YEARS, "DssVest/tau-too-long");
    require(_eta <= _tau, "DssVest/eta-too-long");
    require(ids < type(uint256).max, "DssVest/ids-overflow");

    id = ++ids;
    awards[id] = Award({
        usr: _usr,
        bgn: toUint48(_bgn),
        clf: toUint48(add(_bgn, _eta)),
        fin: toUint48(add(_bgn, _tau)),
        tot: toUint128(_tot),
        rxd: 0,
        mgr: _mgr,
        res: 0
    });
    emit Init(id, _usr);
}
```
This function is used to create a new vesting award, with various time checks to validate input values, and assigns a unique ID for each vesting contract created.

#### Claim Calculation Functions

The following functions manage the actual release of vested tokens by calculating the amount claimable at any given time.
```solidity
function accrued(uint256 _id, uint256 _end) public view returns (uint256 amt) {
    Award memory _award = awards[_id];
    if (block.timestamp < _award.bgn) return 0;
    amt = mul(_award.tot, min(sub(_end, _award.bgn), sub(_award.fin, _award.bgn))) / sub(_award.fin, _award.bgn);
}
```
- This accrued function calculates the total amount of tokens that should have vested by a specified end time (_end), based on the award’s parameters.
- It ensures that no tokens vest before the vesting period (_bgn).
- Uses the formula for linear vesting based on the elapsed time to calculate the vested amount at any point within the start (_bgn) and end (_fin) of the vesting schedule.

```solidity
function unpaid(uint256 _id) public view returns (uint256 amt) {
    Award memory _award = awards[_id];
    amt = sub(accrued(_id, block.timestamp), _award.rxd);
}
```
- function unpaid returns the amount of tokens that are vested but have not yet been claimed (accrued - rxd), where rxd tracks already claimed tokens.
- This is useful for calculating how many tokens a user can claim immediately.

#### Claiming Vesting Tokens
```solidity
function vest(uint256 _id) external lock {
    Award memory _award = awards[_id];
    require(_award.usr != address(0), "DssVest/not-valid-award");
    require(block.timestamp >= _award.clf, "DssVest/cliff-not-reached");

    uint256 amt = unpaid(_id);
    require(amt > 0, "DssVest/no-amount-to-vest");

    awards[_id].rxd = toUint128(add(_award.rxd, amt));
    _vest(_award.usr, amt);
    emit Vest(_id, amt);
}
```
- vest allows a user to claim their vested tokens if the cliff period (clf) has passed.
- It calculates the unclaimed amount (unpaid), updates rxd with the claimed amount, and calls _vest to transfer the tokens.
- Emits a Vest event to log the claimed amount.

#### Restrict and Unrestrict Functions
```solidity
function restrict(uint256 _id) external lock auth {
    require(awards[_id].usr != address(0), "DssVest/not-valid-award");
    awards[_id].res = 1;
    emit Restrict(_id);
}
```
- restrict allows authorized users to enforce restrictions on a specific vesting award by setting its res value to 1.
- This can be used to temporarily prevent a user from claiming tokens, with an emitted Restrict event indicating the action.

```solidity
function unrestrict(uint256 _id) external lock auth {
    require(awards[_id].usr != address(0), "DssVest/not-valid-award");
    awards[_id].res = 0;
    emit Unrestrict(_id);
}
```
This function removes restrictions on an award, allowing the recipient to claim tokens if conditions are met.

#### Revoking (Yanking) Vesting Awards

```solidity
function yank(uint256 _id, uint256 _end) external lock {
    Award memory _award = awards[_id];
    require(_award.mgr == msg.sender || wards[msg.sender] == 1, "DssVest/not-authorized-manager");
    require(_award.fin > _end, "DssVest/not-after-end");
    require(block.timestamp < _award.fin, "DssVest/already-ended");

    awards[_id].fin = toUint48(_end);
    emit Yank(_id, _end);
}
```
- The yank function shortens an award’s vesting period by changing the end time (fin) to _end.
- Only the vesting manager or an authorized user can call it.
- The Yank event signals this action, helping track changes in vesting schedules.

#### Award Transferability
```solidity
function move(uint256 _id, address _dst) external lock {
    Award memory _award = awards[_id];
    require(_award.usr == msg.sender, "DssVest/not-authorized-award-recipient");
    require(_dst != address(0), "DssVest/invalid-destination");

    awards[_id].usr = _dst;
    emit Move(_id, _dst);
}
```
- move enables the current award recipient to transfer their vesting award to another address.
- Only the award recipient (usr) can execute this action.
- Emits a Move event with the ID and new address for transparency.

## Advanced Vesting Contract Types
DssVest is a base contract, while its derivatives provide specific vesting functionalities for different use cases in the MakerDAO ecosystem.

#### DssVestMintable - Mintable Vesting Contract
This derived contract enables minting tokens upon vesting, leveraging the MintLike interface.

```solidity
contract DssVestMintable is DssVest {
    MintLike public immutable gem;

    constructor(address _gem) public DssVest() {
        gem = MintLike(_gem);
    }

    function _vest(address _usr, uint256 _amt) internal override {
        gem.mint(_usr, _amt);
    }
}
```
- DssVestMintable uses the MintLike interface to mint tokens directly to the recipient when _vest is called.
- This setup is appropriate for tokens that are mintable and do not rely solely on pre-existing token balances.

#### DssVestSuckable - DAI Vesting Contract
This variant creates DAI vesting schedules, allowing the core Maker vat system to mint DAI to the recipient.
```solidity
contract DssVestSuckable is DssVest {
    VatLike     public immutable vat;
    DaiJoinLike public immutable daiJoin;
    address     public immutable vow;

    constructor(address _vat, address _daiJoin, address _vow) public DssVest() {
        vat = VatLike(_vat);
        daiJoin = DaiJoinLike(_daiJoin);
        vow = _vow;
        VatLike(_vat).hope(_daiJoin);
    }

    function _vest(address _usr, uint256 _amt) internal override {
        vat.suck(vow, address(this), mul(_amt, 1e27));
        daiJoin.exit(_usr, _amt);
    }
}

```
This contract uses the VatLike and DaiJoinLike interfaces to mint DAI and send it to the user.
_vest calls suck on the vat to mint DAI equivalent to the vested amount, exiting it via daiJoin.

#### DssVestTransferrable - Transferable Token Vesting Contract
This version utilizes pre-existing token balances for vesting by transferring tokens from the contract’s balance to the recipient.

```solidity
contract DssVestTransferrable is DssVest {
    TokenLike public immutable gem;

    constructor(address _gem) public DssVest() {
        gem = TokenLike(_gem);
    }

    function _vest(address _usr, uint256 _amt) internal override {
        require(gem.transferFrom(address(this), _usr, _amt), "DssVestTransferrable/failed-transfer");
    }
}
```
- DssVestTransferrable interacts with standard ERC20 tokens, relying on the contract’s balance to vest tokens by transferring them to the user.
- _vest uses transferFrom to execute the token transfer, making it suitable for non-mintable, pre-minted token balances.

## Use Cases
This code could be used for:

- Token Distribution: Ensuring a gradual release of tokens to early investors, founders, or team members.
- Employee Compensation: Rewarding employees with tokens that are distributed over time to encourage long-term commitment.
- Grant Programs: Distributing tokens to contributors or developers over a set period as a reward for their work.

### Summary
- Base Contract (DssVest): Sets up core vesting operations, handling awards and claims.
- Mintable Vesting (DssVestMintable): Mints tokens directly.
- DAI Vesting (DssVestSuckable): Mints DAI through the Maker protocol.
- Transferrable Vesting (DssVestTransferrable): Transfers pre-existing tokens to the recipient.
These contracts implement flexible vesting mechanisms, ensuring tokens are distributed according to a pre-defined schedule with optional restrictions, management, and revocability.
