---
title: "SushiSwap(Polygon)のFarm実装調査メモ(MiniChefV2.sol)"
emoji: "👨‍🌾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["solidity"]
published: true
---

# コントラクト
https://polygonscan.com/address/0x0769fd68dfb93167989c6f7254cd0d766fb2841f#code

## Compiler Version
- v0.6.12+commit.27d51765

## アウトライン

:::details library
```solidity
library BoringMath {...}
library BoringMath128 {...}
library BoringMath64 {...}
library BoringMath32 {...}
library BoringERC20 {...}
library SignedSafeMath { ... }
```
:::

:::details interface
```solidity
interface IERC20 {...}
interface IMasterChef {...}

interface IMigratorChef {
    function migrate(IERC20 token) external returns (IERC20);
}

interface IRewarder {
    using BoringERC20 for IERC20;
    function onSushiReward(uint256 pid, address user, address recipient, uint256 sushiAmount, uint256 newLpAmount) external;
    function pendingTokens(uint256 pid, address user, uint256 sushiAmount) external view returns (IERC20[] memory, uint256[] memory);
}
```
- IMasterChefはコントラクト内で利用されていない
:::

:::details contract
```solidity
contract BaseBoringBatchable {...}
contract BoringBatchable is BaseBoringBatchable {...}
contract BoringOwnableData {...}
contract BoringOwnable is BoringOwnableData {...}

contract MiniChefV2 is BoringOwnable, BoringBatchable {
    struct UserInfo {
        uint256 amount;
        int256 rewardDebt;
    }

    struct PoolInfo {
        uint128 accSushiPerShare;
        uint64 lastRewardTime;
        uint64 allocPoint;
    }

    function poolLength() public view returns (uint256 pools) {...}

    function add(uint256 allocPoint, IERC20 _lpToken, IRewarder _rewarder) public onlyOwner {...}
    function set(uint256 _pid, uint256 _allocPoint, IRewarder _rewarder, bool overwrite) public onlyOwner {...}

    function setSushiPerSecond(uint256 _sushiPerSecond) public onlyOwner {...}
    function setMigrator(IMigratorChef _migrator) public onlyOwner {...}

    function migrate(uint256 _pid) public {...}
    function pendingSushi(uint256 _pid, address _user) external view returns (uint256 pending) {...}
    function massUpdatePools(uint256[] calldata pids) external {...}

    function updatePool(uint256 pid) public returns (PoolInfo memory pool) {...}
    function deposit(uint256 pid, uint256 amount, address to) public {...}
    function withdraw(uint256 pid, uint256 amount, address to) public {...}
    function harvest(uint256 pid, address to) public {...}
    function withdrawAndHarvest(uint256 pid, uint256 amount, address to) public {...}
    function emergencyWithdraw(uint256 pid, address to) public {...}
}
```
:::


# ユースケースごとの処理詳細
## 新しいプールを作成する(MiniChefV2.add)
:::details add
```solidity
/// @notice Add a new LP to the pool. Can only be called by the owner.
/// DO NOT add the same LP token more than once. Rewards will be messed up if you do.
/// @param allocPoint AP of the new pool.
/// @param _lpToken Address of the LP ERC-20 token.
/// @param _rewarder Address of the rewarder delegate.
function add(uint256 allocPoint, IERC20 _lpToken, IRewarder _rewarder) public onlyOwner {
    totalAllocPoint = totalAllocPoint.add(allocPoint);
    lpToken.push(_lpToken);
    rewarder.push(_rewarder);

    poolInfo.push(PoolInfo({
        allocPoint: allocPoint.to64(),
        lastRewardTime: block.timestamp.to64(),
        accSushiPerShare: 0
    }));
    emit LogPoolAddition(lpToken.length.sub(1), allocPoint, _lpToken, _rewarder);
}
```
:::
1. トータルAP(アロケーションポイント)に加算する
1. プール情報を保存する


## LPトークンを預ける (StakingRewards.deposit)
:::details deposit
```solidity
/// @notice Deposit LP tokens to MCV2 for SUSHI allocation.
/// @param pid The index of the pool. See `poolInfo`.
/// @param amount LP token amount to deposit.
/// @param to The receiver of `amount` deposit benefit.
function deposit(uint256 pid, uint256 amount, address to) public {
    PoolInfo memory pool = updatePool(pid);
    UserInfo storage user = userInfo[pid][to];

    // Effects
    user.amount = user.amount.add(amount);
    user.rewardDebt = user.rewardDebt.add(int256(amount.mul(pool.accSushiPerShare) / ACC_SUSHI_PRECISION));

    // Interactions
    IRewarder _rewarder = rewarder[pid];
    if (address(_rewarder) != address(0)) {
        _rewarder.onSushiReward(pid, to, to, 0, user.amount);
    }

    lpToken[pid].safeTransferFrom(msg.sender, address(this), amount);

    emit Deposit(msg.sender, pid, amount, to);
}
```
:::
1. プール情報を更新する
1. ユーザの残高等報酬計算用の情報を更新する
1. Rewarderが設定されている場合Rewarderを呼び出す
1. LPトークンをユーザからコントラクトに送る


## ステーク報酬を取得する (MiniChefV2.harvest)
:::details harvest
```solidity
/// @notice Harvest proceeds for transaction sender to `to`.
/// @param pid The index of the pool. See `poolInfo`.
/// @param to Receiver of SUSHI rewards.
function harvest(uint256 pid, address to) public {
    PoolInfo memory pool = updatePool(pid);
    UserInfo storage user = userInfo[pid][msg.sender];
    int256 accumulatedSushi = int256(user.amount.mul(pool.accSushiPerShare) / ACC_SUSHI_PRECISION);
    uint256 _pendingSushi = accumulatedSushi.sub(user.rewardDebt).toUInt256();

    // Effects
    user.rewardDebt = accumulatedSushi;

    // Interactions
    if (_pendingSushi != 0) {
        SUSHI.safeTransfer(to, _pendingSushi);
    }
    
    IRewarder _rewarder = rewarder[pid];
    if (address(_rewarder) != address(0)) {
        _rewarder.onSushiReward( pid, msg.sender, to, _pendingSushi, user.amount);
    }

    emit Harvest(msg.sender, pid, _pendingSushi);
}
```
:::
1. プール情報を更新する
1. 報酬を計算する
1. ユーザの残高等報酬計算用の情報を更新する
1. 報酬をコントラクトからユーザに送る
1. Rewarderが設定されている場合Rewarderを呼び出す


## LPトークンを引き出す (MiniChefV2.withdraw)
:::details withdraw
```solidity
/// @notice Withdraw LP tokens from MCV2.
/// @param pid The index of the pool. See `poolInfo`.
/// @param amount LP token amount to withdraw.
/// @param to Receiver of the LP tokens.
function withdraw(uint256 pid, uint256 amount, address to) public {
    PoolInfo memory pool = updatePool(pid);
    UserInfo storage user = userInfo[pid][msg.sender];

    // Effects
    user.rewardDebt = user.rewardDebt.sub(int256(amount.mul(pool.accSushiPerShare) / ACC_SUSHI_PRECISION));
    user.amount = user.amount.sub(amount);

    // Interactions
    IRewarder _rewarder = rewarder[pid];
    if (address(_rewarder) != address(0)) {
        _rewarder.onSushiReward(pid, msg.sender, to, 0, user.amount);
    }
    
    lpToken[pid].safeTransfer(to, amount);

    emit Withdraw(msg.sender, pid, amount, to);
}
```
:::
1. プール情報を更新する
1. ユーザの残高等報酬計算用の情報を更新する
1. Rewarderが設定されている場合Rewarderを呼び出す
1. LPトークンをコントラクトからユーザに送る


## LPトークンと報酬を引き出す (MiniChefV2.withdrawAndHarvest)
:::details withdrawAndHarvest
```solidity
    /// @notice Withdraw LP tokens from MCV2 and harvest proceeds for transaction sender to `to`.
    /// @param pid The index of the pool. See `poolInfo`.
    /// @param amount LP token amount to withdraw.
    /// @param to Receiver of the LP tokens and SUSHI rewards.
    function withdrawAndHarvest(uint256 pid, uint256 amount, address to) public {
        PoolInfo memory pool = updatePool(pid);
        UserInfo storage user = userInfo[pid][msg.sender];
        int256 accumulatedSushi = int256(user.amount.mul(pool.accSushiPerShare) / ACC_SUSHI_PRECISION);
        uint256 _pendingSushi = accumulatedSushi.sub(user.rewardDebt).toUInt256();

        // Effects
        user.rewardDebt = accumulatedSushi.sub(int256(amount.mul(pool.accSushiPerShare) / ACC_SUSHI_PRECISION));
        user.amount = user.amount.sub(amount);
        
        // Interactions
        SUSHI.safeTransfer(to, _pendingSushi);

        IRewarder _rewarder = rewarder[pid];
        if (address(_rewarder) != address(0)) {
            _rewarder.onSushiReward(pid, msg.sender, to, _pendingSushi, user.amount);
        }

        lpToken[pid].safeTransfer(to, amount);

        emit Withdraw(msg.sender, pid, amount, to);
        emit Harvest(msg.sender, pid, _pendingSushi);
    }
```
:::
1. プール情報を更新する
1. 報酬を計算する
1. ユーザの残高等報酬計算用の情報を更新する
1. 報酬をコントラクトからユーザに送る
1. Rewarderが設定されている場合Rewarderを呼び出す
1. LPトークンをコントラクトからユーザに送る


# 報酬の仕組み
:::details updatePool
```solidity
/// @notice Update reward variables of the given pool.
/// @param pid The index of the pool. See `poolInfo`.
/// @return pool Returns the pool that was updated.
function updatePool(uint256 pid) public returns (PoolInfo memory pool) {
    pool = poolInfo[pid];
    if (block.timestamp > pool.lastRewardTime) {
        uint256 lpSupply = lpToken[pid].balanceOf(address(this));
        if (lpSupply > 0) {
            uint256 time = block.timestamp.sub(pool.lastRewardTime);
            uint256 sushiReward = time.mul(sushiPerSecond).mul(pool.allocPoint) / totalAllocPoint;
            pool.accSushiPerShare = pool.accSushiPerShare.add((sushiReward.mul(ACC_SUSHI_PRECISION) / lpSupply).to128());
        }
        pool.lastRewardTime = block.timestamp.to64();
        poolInfo[pid] = pool;
        emit LogUpdatePool(pid, pool.lastRewardTime, lpSupply, pool.accSushiPerShare);
    }
}
```
:::
- 報酬分配のコアとなる考え方はStakingRewardsの時と同じ

https://zenn.dev/sakuracase/articles/9a6f6e33d6326c#%E7%8D%B2%E5%BE%97%E5%A0%B1%E9%85%AC%E9%A1%8D%E6%9B%B4%E6%96%B0%E3%81%AE%E4%BB%95%E7%B5%84%E3%81%BF(stakingrewards.updatereward)

- StakingRewardsは事前にトータル報酬額と配布期間を設定して秒間報酬額を算出するが、MiniChefでは秒間報酬額(全プール合計)のみを直接設定する
    - そのためMiniChefではコントラクト内の報酬が足りなくなり報酬を受け取れなくなる可能性がある
- MiniChefでは複数のステーキングプールを作成可能となっている。各ステーキングプールにアロケーションポイント（AP）が設定されており、そのプールの報酬はAPに応じて変わる
    - 秒間報酬 * 対象プールのAP / トータルAP

- LPトークンのステーク総量を`lpToken[pid].balanceOf(address(this));`で取得しているため、add関数のコメントにあるように同じLPトークンのプールを作成すると報酬配布量がおかしくなる(少なくなる)


# Rewarderについて
- Sushi以外の報酬として別のトークンを報酬として設定可能

https://dev.sushi.com/sushiswap/contracts/masterchefv2/adding-double-incentives

- 執筆時点のPolygonのFarmでは[ComplexRewarderTime](https://polygonscan.com/address/0xa3378Ca78633B3b9b2255EAa26748770211163AE#code)が全てのプールのRewarderとして設定されていてMATICを報酬として受け取ることが可能


# 備考
- MiniChefV2と似たコードでMasterChefV2が存在している
    - Mainnetで利用されていてPolygonには存在していない
    - MasterChefV2は秒間の報酬ではなくブロックごとの報酬獲得
    - MasterChefV2はMasterChef(V1)がSUSHIのmint権限を持っていることからMasterChefを呼び出す処理が含まれているが、MiniChefにはない

# 参考
https://dev.sushi.com/sushiswap/contracts/masterchefv2
https://github.com/sushiswap/sushiswap/tree/master