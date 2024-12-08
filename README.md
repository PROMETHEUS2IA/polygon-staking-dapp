# polygon-staking-dapp
DApp de staking en la red Polygon 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Staking {
    IERC20 public stakingToken;

    struct Stake {
        uint256 amount;
        uint256 startTime;
        uint256 lockPeriod; // En segundos (30 días o 365 días)
    }

    mapping(address => Stake) public stakes;
    uint256 public constant MIN_MONTHLY_REWARD = 3;  // 3%
    uint256 public constant MIN_ANNUAL_REWARD = 40; // 40%

    constructor(address _token) {
        stakingToken = IERC20(_token); // Dirección del token ERC-20
    }

    function stake(uint256 _amount, uint256 _lockPeriod) external {
        require(_amount > 0, "Stake amount must be greater than 0");
        require(
            _lockPeriod == 30 days || _lockPeriod == 365 days,
            "Invalid lock period"
        );
        require(stakes[msg.sender].amount == 0, "Already staking");

        // Transfiere tokens al contrato
        stakingToken.transferFrom(msg.sender, address(this), _amount);

        // Registra el stake
        stakes[msg.sender] = Stake(_amount, block.timestamp, _lockPeriod);
    }

    function calculateReward(address _staker) public view returns (uint256) {
        Stake memory userStake = stakes[_staker];
        require(userStake.amount > 0, "No active stake");

        uint256 elapsedTime = block.timestamp - userStake.startTime;
        uint256 rewardRate = userStake.lockPeriod == 30 days
            ? MIN_MONTHLY_REWARD
            : MIN_ANNUAL_REWARD;

        // Calcula recompensa mínima proporcional al tiempo transcurrido
        uint256 reward = (userStake.amount * rewardRate * elapsedTime) /
            (userStake.lockPeriod * 100);

        return reward;
    }

    function withdraw() external {
        Stake memory userStake = stakes[msg.sender];
        require(userStake.amount > 0, "No active stake");
        require(
            block.timestamp >= userStake.startTime + userStake.lockPeriod,
            "Stake is still locked"
        );

        // Calcula recompensa
        uint256 reward = calculateReward(msg.sender);

        // Borra el registro del stake
        delete stakes[msg.sender];

        // Devuelve tokens al usuario
        stakingToken.transfer(msg.sender, userStake.amount + reward);
    }
}
address public owner;

constructor(address _token) {
    stakingToken = IERC20(_token); 
    owner = msg.sender; // Establece al creador como dueño
}

modifier onlyOwner() {
    require(msg.sender == owner, "Only owner can adjust rewards");
    _;
}

function setRewardRate(uint256 _monthlyRate, uint256 _annualRate) external onlyOwner {
    MIN_MONTHLY_REWARD = _monthlyRate;
    MIN_ANNUAL_REWARD = _annualRate;
}
function getStakeDetails(address _staker) external view returns (uint256 amountStaked, uint256 reward) {
    Stake memory userStake = stakes[_staker];
    require(userStake.amount > 0, "No active stake");
    
    uint256 accumulatedReward = calculateReward(_staker);
    return (userStake.amount, accumulatedReward);
}
function getContractBalance() external view returns (uint256) {
    return stakingToken.balanceOf(address(this));
}
