# Launchpad Smart Contract Documentation

## What is this?

This is a token presale platform where projects can launch their token sales. Think of it like Kickstarter, but for crypto tokens. Projects set up their sale, users buy tokens, and everything happens on-chain with built-in safety features.

**Built with:** Solidity 0.8.22, OpenZeppelin libraries  
**Chain:** Ethereum-compatible networks

---

## Main Features

### Token Sales
Projects can create customized token presales with their own rules - pricing, time limits, minimum/maximum purchase amounts. We support any ERC20 token as payment (USDT, USDC, etc.).

### Vesting System
Not all tokens need to be released immediately. We have a flexible vesting system where:
- Some tokens unlock at TGE (Token Generation Event)
- The rest unlock gradually over time in cycles
- Users claim their unlocked tokens when ready

### Admin Controls
The platform has role-based permissions. Different admins can approve sales, pause them, adjust parameters, or manage refunds - whatever permissions they're given.

### Refunds
Projects can enable refunds if needed. Users can get their money back within a specified timeframe, though they need to return any tokens they've already claimed.

### Platform Fees
There's a default 5% platform fee, but this can be customized per launchpad if needed.

---

## How It Works

### 1. Creating a Launchpad

Anyone can create a presale by calling `addLaunchpad()` with these details:

- Which token you're selling
- Token price (how many tokens per payment token)
- Which payment token to accept
- When the sale starts and ends
- Min/max purchase limits
- Total tokens available
- Vesting setup (if you want tokens locked)

The contract validates everything, transfers your tokens into escrow, and sets the status to "Pending" until an admin approves it.

**Quick example:**
```
Token price: 10 (means 10 tokens per 1 USDT)
Payment: USDT
Duration: Dec 1 - Feb 1
Min buy: 100 USDT
Max buy: 10,000 USDT
Supply: 1 million tokens
```

### 2. Buying Tokens

Once approved, users call `buyTokens()` with the amount they want to spend. The contract:

- Checks if the sale is active and not paused
- Verifies the amount is within limits
- Calculates how many tokens they get
- Takes their payment
- Either sends tokens immediately OR records them for vesting
- Updates all the stats

The math is straightforward: `tokens = (payment × tokenPrice) / paymentTokenDecimals`

### 3. Claiming Vested Tokens

If vesting is enabled, users need to claim their tokens as they unlock. They call `unlock()` with their purchase ID.

The contract calculates what's available based on:
- TGE unlock (e.g., 20% immediately)
- Time passed since TGE
- Vesting cycle (e.g., 10% every 30 days)

**Example scenario:**
```
Bought: 10,000 tokens
TGE: 20% unlocks = 2,000 tokens
Cycles: 10% every month
After 3 months: 3 more cycles × 1,000 = 3,000 tokens
Total unlocked: 5,000 tokens
Already claimed: 2,000 tokens
Can claim now: 3,000 tokens
```

### 4. Getting Refunds

If the project enables refunds, users can call `requestRefund()` before the deadline. The contract:

- Checks if refunds are still open
- Calculates how much to return (only for unlocked tokens if vesting)
- Returns the payment
- Takes back any claimed tokens
- Updates all counters

This protects both sides - users can exit if needed, but can't refund tokens they haven't unlocked yet.

### 5. Withdrawing Funds

After the sale ends, the project owner calls `withdraw()` to collect the funds. The contract:

- Returns any unsold tokens
- Deducts the platform fee (default 5%)
- Sends the rest to the owner
- Marks everything as complete

---

## Admin Functions

Admins have specific permissions set by the contract owner. Here's what they can do:

**Approve/Reject Sales**
- `setLaunchpadApproval()` - greenlight a pending sale
- `rejectLaunchpadProject()` - reject it and return tokens

**Emergency Controls**
- `setLaunchpadPause()` - pause/unpause buying (for emergencies)

**Update Parameters**
- `setTokenPrice()` - change the token price
- `setPaymentToken()` - switch payment token
- `setMinMaxBuy()` - adjust purchase limits
- `setPresaleTime()` - modify sale duration
- `setTotalTokensOffered()` - add more tokens to the sale

**Vesting Configuration**
- `setTGE()` - change TGE time and unlock percentage
- `setVestingCycle()` - adjust vesting schedule

**Refund Management**
- `setRefundStatus()` - enable/disable refunds
- `updateRefundEndTime()` - change refund deadline

**Fee Settings**
- `setGlobalFee()` - change platform-wide fee
- `setLaunchpadFee()` - set custom fee for specific sale
- `changeFeesTakes()` - update fee recipient

---

## Key Events

The contract emits events so you can track everything:

- `LaunchpadAdded` - new sale created
- `LaunchpadApproval` - status changed (approved/rejected/ended)
- `UserDetailsUpdated` - someone bought tokens
- `TokensClaimed` - user claimed vested tokens
- `TokensRefunded` - refund processed
- Plus various events for parameter updates

---

## Important Notes

**Security:**
- Uses OpenZeppelin's battle-tested libraries
- ReentrancyGuard prevents double-spending attacks
- SafeERC20 for safe token transfers
- All admin actions are permission-gated

**Validation:**
- Everything is checked before execution
- Invalid parameters are rejected immediately
- Time-based checks prevent early/late actions
- Math is done carefully to avoid overflows

**Flexibility:**
- Works with any ERC20 token
- Vesting is optional
- Refunds are optional
- Fees are configurable
- Each sale can have unique parameters

**User Experience:**
- Users can track their purchases and vesting schedule
- Clear error messages when things go wrong
- Events make everything trackable on-chain
- Simple claim process for vested tokens

---

## Common Questions

**Q: What happens to unsold tokens?**  
They're returned to the project owner when they withdraw.

**Q: Can parameters change after approval?**  
Yes, admins can update most parameters, but it requires proper permissions.

**Q: How are fees calculated?**  
As a percentage of total raised. Default is 5%, but can be customized.

**Q: What if I want to refund after claiming some tokens?**  
You can only refund what you've unlocked/claimed. The rest stays vested.

**Q: Can a sale be paused?**  
Yes, admins can pause in emergencies. Users can't buy while paused.

---

That's the overview. The contract handles all the complex logic so projects can focus on their token sale and users can participate safely.
