
# TRUSTAI Token Lock Contract - Technical Documentation

## Introduction

Hey there! This document covers everything you need to know about the TRUSTAI_LOCK smart contract. I've tried to make it as straightforward as possible while covering all the important bits.

The contract basically does one thing really well: it locks tokens (both regular ERC20 tokens and liquidity pool tokens) for a specified time period. You can do simple time-locked releases or more complex vesting schedules where tokens unlock gradually over time.

**Contract Details:**
- Solidity Version: ^0.8.20
- License: MIT
- Main Dependencies: OpenZeppelin (SafeERC20, EnumerableSet, Address)

---

## What Can This Contract Do?

### Main Features

**1. Regular Token Locks**
- Lock any ERC20 token until a specific unlock date
- Works like a time-locked safe - once you lock them, they stay locked until the date you specified
  
**2. Vesting Locks**
- More different than regular locks
- Tokens unlock gradually over time in cycles
- You can set an initial unlock at TGE (Token Generation Event), then have the rest unlock in periodic cycles

**3. Liquidity Pool (LP) Token Support**
- Specifically designed to handle Uniswap V2 style LP tokens
- Tracks which tokens are LP tokens vs normal tokens

**4. Bulk Locking**
- Need to lock tokens for multiple addresses at once? No problem
- The `multipleVestingLock` function lets you create locks for many addresses in a single transaction
- Super useful for airdrops or team allocations

**5. Lock Management**
- Edit your locks (increase amount, extend time)
- Transfer lock ownership to another address
- Add descriptions to your locks for better organization
- Renounce ownership if needed

---

## How It Works - The Technical Stuff

## Creating Locks

### Option 1: Regular Time Lock

This is the simplest option - lock tokens until a specific date.

```solidity
function lock(
    address owner,
    address token,
    bool isLpToken,
    uint256 amount,
    uint256 unlockDate,
    string memory description
) external returns (uint256 id)
```

**Parameters:**
- `owner` - Who will own this lock (can be different from caller)
- `token` - Address of the token to lock
- `isLpToken` - Set to `true` if this is an LP token, `false` otherwise
- `amount` - How many tokens to lock
- `unlockDate` - Unix timestamp when tokens can be unlocked
- `description` - Optional note about this lock (e.g., "Team allocation")

**Requirements:**
- Token address must be valid (not zero address)
- Amount must be greater than 0
- Unlock date must be in the future
- You need to approve the contract to spend your tokens first

**What happens:**
1. Contract validates all your inputs
2. Creates a new Lock record with an ID
3. Transfers tokens from you to the contract
4. Adds the lock to tracking mappings
5. Returns the lock ID


### Option 2: Vesting Lock

For gradual unlocking over time.

```solidity
function vestingLock(
    address owner,
    address token,
    bool isLpToken,
    uint256 amount,
    uint256 tgeDate,
    uint256 tgeBps,
    uint256 cycle,
    uint256 cycleBps,
    string memory description
) external returns (uint256 id)
```

**Additional vesting parameters:**
- `tgeDate` - When the first unlock happens (Token Generation Event)
- `tgeBps` - What percentage unlocks at TGE (in basis points)
- `cycle` - How long between each unlock after TGE (in seconds)
- `cycleBps` - What percentage unlocks per cycle (in basis points)

**Requirements:**
- All the same requirements as regular lock
- TGE date must be in the future
- Cycle must be greater than 0
- Both tgeBps and cycleBps must be between 0 and 10,000 (0-100%)
- tgeBps + cycleBps must not exceed 10,000 (can't unlock more than 100% total)

**Important note:** The sum of TGE percentage and cycle percentage tells the contract how to distribute unlocks. If they don't add up to 100%, the remaining percentage will unlock in the final cycle.

### Option 3: Multiple Vesting Locks

Need to create the same vesting schedule for multiple addresses? This function's got you covered.

```solidity
function multipleVestingLock(
    address[] calldata owners,
    uint256[] calldata amounts,
    address token,
    bool isLpToken,
    uint256 tgeDate,
    uint256 tgeBps,
    uint256 cycle,
    uint256 cycleBps,
    string memory description
) external returns (uint256[] memory)
```

**important points:**
- `owners` and `amounts` arrays must be the same length
- Each address gets its own lock with the same vesting schedule but potentially different amounts
- More gas-efficient than creating locks individually
- Contract transfers the total of all amounts in one go



## Unlocking Tokens

### For Regular Locks

```solidity
function unlock(uint256 lockId) external
```

Pretty straightforward - once the unlock date has passed, call this function with your lock ID and you'll get your tokens back.

**What happens:**
1. Contract checks that you own the lock
2. Verifies the unlock date has passed
3. Makes sure you haven't already unlocked
4. Transfers all tokens to you
5. Removes the lock from tracking (except the main record, which stays for history)
6. Emits a `LockRemoved` event

### For Vesting Locks

Same function, different behavior! The contract automatically detects it's a vesting lock and calculates how much you can withdraw.

**Important:** You can call `unlock()` multiple times on a vesting lock! Each time you call it, you get whatever new tokens have vested since your last withdrawal.

**Checking before you unlock:**

There's a handy view function to see how much you can currently withdraw:

```solidity
function withdrawableTokens(uint256 lockId) external view returns (uint256)
```

This doesn't cost any gas and tells you exactly how many tokens are ready to claim.

---

## Managing Your Locks

### Editing a Lock

```solidity
function editLock(
    uint256 lockId,
    uint256 newAmount,
    uint256 newUnlockDate
) external
```

You can modify your lock, but with some restrictions to maintain security:

**Why these restrictions?**
They prevent people from gaming the system by shortening lock periods or removing tokens after committing to a lock schedule.


### Adding/Changing Description

```solidity
function editLockDescription(uint256 lockId, string memory description) external
```

Simple function to update the description of your lock. Useful for keeping track of what each lock is for.

### Transferring Ownership

```solidity
function transferLockOwnership(uint256 lockId, address newOwner) public
```

Transfer your lock to someone else. Once transferred:
- The new owner can unlock the tokens when they vest
- The new owner can edit the lock
- You lose all control over it

**Use cases:**
- Selling a vested position
- Transferring team allocations to new team members
- Corporate restructuring

**Special case - Renouncing:**
```solidity
function renounceLockOwnership(uint256 lockId) external
```

This transfers ownership to the zero address, effectively burning the lock. The tokens become unrecoverable. Use with extreme caution!

---

## LP Token Handling

The contract has special logic for handling liquidity pool tokens from Uniswap V2 style DEXes.

### How LP Token Detection Works

When you create a lock with `isLpToken = true`, the contract:

1. Calls the `factory()` function on the token address
2. If it gets a valid factory address back, it continues
3. Gets token0 and token1 from the LP pair
4. Verifies with the factory that this pair actually exists
5. Checks that the factory's pair address matches the token address

**Why all this validation?**
To prevent people from gaming the system by claiming a regular token is an LP token, or providing fake LP tokens.

```solidity
function _parseFactoryAddress(address token) internal view returns (address) {
    // Tries to call factory() on the token
    try IUniswapV2Pair(token).factory() returns (address factory) {
        possibleFactoryAddress = factory;
    } catch {
        revert("This token is not a LP token");
    }
    
    require(
        possibleFactoryAddress != address(0) &&
        _isValidLpToken(token, possibleFactoryAddress),
        "This token is not a LP token."
    );
    return possibleFactoryAddress;
}
```


## Querying Lock Information

The contract provides tons of view functions for getting information. Here are the most useful ones:

### Individual Lock Queries

```solidity
// Get a specific lock by ID
function getLockById(uint256 lockId) public view returns (Lock memory)

// Get a lock by array index (more gas efficient for iteration)
function getLockAt(uint256 index) external view returns (Lock memory)

// Get total number of locks ever created
function getTotalLockCount() external view returns (uint256)
```

### User-Specific Queries

```solidity
// Get all LP locks for a user
function lpLocksForUser(address user) external view returns (Lock[] memory)

// Get all normal token locks for a user
function normalLocksForUser(address user) external view returns (Lock[] memory)

// Get counts
function lpLockCountForUser(address user) public view returns (uint256)
function normalLockCountForUser(address user) public view returns (uint256)
function totalLockCountForUser(address user) external view returns (uint256)

// Get specific lock for user by index
function lpLockForUserAtIndex(address user, uint256 index) external view returns (Lock memory)
function normalLockForUserAtIndex(address user, uint256 index) external view returns (Lock memory)
```

### Token-Specific Queries

```solidity
// Get all locks for a specific token
function getLocksForToken(address token, uint256 start, uint256 end) 
    public view returns (Lock[] memory)

// Get total locks for a token
function totalLockCountForToken(address token) external view returns (uint256)
```

### Platform-Wide Statistics

```solidity
// Total unique tokens with locks
function totalTokenLockedCount() external view returns (uint256)
function allLpTokenLockedCount() public view returns (uint256)
function allNormalTokenLockedCount() public view returns (uint256)

// Get cumulative lock info for tokens
function getCumulativeLpTokenLockInfo(uint256 start, uint256 end) 
    external view returns (CumulativeLockInfo[] memory)
    
function getCumulativeNormalTokenLockInfo(uint256 start, uint256 end) 
    external view returns (CumulativeLockInfo[] memory)
```

The `CumulativeLockInfo` struct shows:
```solidity
struct CumulativeLockInfo {
    address token;      // Token address
    address factory;    // Factory address (0x0 for normal tokens)
    uint256 amount;     // Total amount currently locked
}
```

This is super useful for showing "Total Value Locked" statistics on your platform.

---

## Events

The contract emits events for all important actions:

```solidity
// When a new lock is created
event LockAdded(
    uint256 indexed id,
    address token,
    address owner,
    uint256 amount,
    uint256 unlockDate,
    bool isLpToken
);

// When lock parameters are changed
event LockUpdated(
    uint256 indexed id,
    address token,
    address owner,
    uint256 newAmount,
    uint256 newUnlockDate
);

// When a normal lock is fully unlocked
event LockRemoved(
    uint256 indexed id,
    address token,
    address owner,
    uint256 amount,
    uint256 unlockedAt
);

// When tokens are claimed from a vesting lock
event LockVested(
    uint256 indexed id,
    address token,
    address owner,
    uint256 amount,           // Amount claimed this time
    uint256 remaining,        // Amount still locked
    uint256 timestamp
);

// When lock description changes
event LockDescriptionChanged(uint256 lockId);

// When lock ownership transfers
event LockOwnerChanged(uint256 lockId, address owner, address newOwner);
```

---


### Building a Frontend

Here's a typical flow for displaying a user's locks:

```javascript
// 1. Get total lock count for user
const totalLocks = await contract.totalLockCountForUser(userAddress);

// 2. Get LP locks
const lpLocks = await contract.lpLocksForUser(userAddress);

// 3. Get normal locks
const normalLocks = await contract.normalLocksForUser(userAddress);

// 4. For each vesting lock, check withdrawable amount
for (let lock of [...lpLocks, ...normalLocks]) {
    if (lock.cycleBps > 0) {  // It's a vesting lock
        const withdrawable = await contract.withdrawableTokens(lock.id);
        // Display withdrawable amount to user
    }
}
```


## Wrapping Up

This contract is pretty solid for token locking needs. The vesting logic is flexible, the LP token support is well-implemented, and the query functions make it easy to build frontends.

