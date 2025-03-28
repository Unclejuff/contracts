# DipNft Contract Mapping Example

This file provides an example AssemblyScript mapping for the DipNft contract events. It shows how to properly handle and index NFT-related events in a subgraph.

## Mapping Code

```typescript
import { BigInt, Address, log } from '@graphprotocol/graph-ts'
import {
  NFTEvent,
  DCURewards,
  DCURewardTriggered,
  Minted,
  NFTUpgraded
} from '../generated/DipNft/DipNft'
import { 
  User, 
  Token, 
  NFTClaim, 
  NFTUpgrade, 
  Global 
} from '../generated/schema'

// Helper function to ensure User entity exists
function getOrCreateUser(address: Address): User {
  let userId = address.toHexString()
  let user = User.load(userId)
  
  if (!user) {
    user = new User(userId)
    user.totalClaims = 0
    user.totalUpgrades = 0
    user.totalRewards = BigInt.fromI32(0)
    user.createdAt = BigInt.fromI32(0)
  }
  
  return user
}

// Helper function to ensure Token entity exists
function getOrCreateToken(tokenId: BigInt, owner: Address): Token {
  let tokenIdHex = tokenId.toString()
  let token = Token.load(tokenIdHex)
  
  if (!token) {
    token = new Token(tokenIdHex)
    token.tokenId = tokenId
    token.owner = owner.toHexString()
    token.level = 1
    token.impactLevel = 1
    token.createdAt = BigInt.fromI32(0)
  }
  
  return token
}

// Helper function to ensure Global entity exists
function getOrCreateGlobal(): Global {
  let global = Global.load('global')
  
  if (!global) {
    global = new Global('global')
    global.totalNFTsClaimed = 0
    global.totalUpgrades = 0
    global.totalRewardsDistributed = BigInt.fromI32(0)
    global.lastUpdated = BigInt.fromI32(0)
  }
  
  return global
}

// Handle the unified NFTEvent event
export function handleNFTEvent(event: NFTEvent): void {
  let user = getOrCreateUser(event.params.user)
  let token = getOrCreateToken(event.params.tokenId, event.params.user)
  let global = getOrCreateGlobal()
  let eventType = event.params.eventType
  
  // Update user's last activity
  user.lastActivity = event.params.timestamp
  
  // Update total rewards
  user.totalRewards = user.totalRewards.plus(event.params.rewardAmount)
  
  // Process based on event type
  if (eventType == "CLAIM") {
    // This is a new NFT claim
    user.totalClaims += 1
    
    // Update token data
    token.createdAt = event.params.timestamp
    
    // Create claim record
    let claimId = event.transaction.hash.toHexString() + '-' + event.logIndex.toString()
    let claim = new NFTClaim(claimId)
    claim.user = user.id
    claim.token = token.id
    claim.timestamp = event.params.timestamp
    claim.rewardAmount = event.params.rewardAmount
    claim.transaction = event.transaction.hash.toHexString()
    
    // Update global stats
    global.totalNFTsClaimed += 1
    global.totalRewardsDistributed = global.totalRewardsDistributed.plus(event.params.rewardAmount)
    global.lastUpdated = event.params.timestamp
    
    // Save claim entity
    claim.save()
    
    // Set user creation time if not set
    if (user.createdAt.equals(BigInt.fromI32(0))) {
      user.createdAt = event.params.timestamp
    }
  } 
  else if (eventType == "UPGRADE") {
    // This is an NFT level upgrade
    user.totalUpgrades += 1
    
    // Update token data
    token.level = event.params.newLevel.toI32()
    token.lastUpgradedAt = event.params.timestamp
    
    // Create upgrade record
    let upgradeId = event.transaction.hash.toHexString() + '-' + event.logIndex.toString()
    let upgrade = new NFTUpgrade(upgradeId)
    upgrade.user = user.id
    upgrade.token = token.id
    upgrade.oldLevel = event.params.oldLevel.toI32()
    upgrade.newLevel = event.params.newLevel.toI32()
    upgrade.timestamp = event.params.timestamp
    upgrade.rewardAmount = event.params.rewardAmount
    upgrade.transaction = event.transaction.hash.toHexString()
    
    // Update global stats
    global.totalUpgrades += 1
    global.totalRewardsDistributed = global.totalRewardsDistributed.plus(event.params.rewardAmount)
    global.lastUpdated = event.params.timestamp
    
    // Save upgrade entity
    upgrade.save()
  }
  
  // Save common entities
  user.save()
  token.save()
  global.save()
}

// Handle the original Minted event (for backward compatibility)
export function handleMinted(event: Minted): void {
  let user = getOrCreateUser(event.params.to)
  let token = getOrCreateToken(event.params.tokenId, event.params.to)
  
  // Set token level from event
  token.level = event.params.nftLevel.toI32()
  
  // Save entities
  user.save()
  token.save()
}

// Handle the original NFTUpgraded event (for backward compatibility)
export function handleNFTUpgraded(event: NFTUpgraded): void {
  let user = getOrCreateUser(event.params.to)
  let token = getOrCreateToken(event.params.tokenId, event.params.to)
  
  // Update token level
  token.level = event.params.newLevel.toI32()
  token.lastUpgradedAt = event.block.timestamp
  
  // Save entities
  user.save()
  token.save()
}

// Handle DCURewards event
export function handleDCURewards(event: DCURewards): void {
  let user = getOrCreateUser(event.params.to)
  
  // Update total rewards
  user.totalRewards = user.totalRewards.plus(event.params.amount)
  user.lastActivity = event.block.timestamp
  
  // Update global stats
  let global = getOrCreateGlobal()
  global.totalRewardsDistributed = global.totalRewardsDistributed.plus(event.params.amount)
  global.lastUpdated = event.block.timestamp
  
  // Save entities
  user.save()
  global.save()
}

// Handle DCURewardTriggered event
export function handleDCURewardTriggered(event: DCURewardTriggered): void {
  let user = getOrCreateUser(event.params.to)
  
  // Update total rewards
  user.totalRewards = user.totalRewards.plus(event.params.amount)
  user.lastActivity = event.block.timestamp
  
  // Update global stats
  let global = getOrCreateGlobal()
  global.totalRewardsDistributed = global.totalRewardsDistributed.plus(event.params.amount)
  global.lastUpdated = event.block.timestamp
  
  // Save entities
  user.save()
  global.save()
}
```

## Usage

This mapping file should be used in conjunction with the DipNft contract ABI and the subgraph configuration. It handles all NFT-related events, including the unified `NFTEvent` for claims and upgrades.

For more details on subgraph development, see The Graph documentation at https://thegraph.com/docs/. 