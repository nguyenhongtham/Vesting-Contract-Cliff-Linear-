# Vesting-Contract-Cliff-Linear-
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract TokenVesting is Ownable {
    IERC20 public token;
    uint256 public totalVested;

    struct VestingSchedule {
        uint256 totalAmount;
        uint256 released;
        uint256 startTime;
        uint256 cliffTime;
        uint256 duration;
    }

    mapping(address => VestingSchedule) public vestings;

    constructor(address _token) {
        token = IERC20(_token);
    }

    function createVesting(address beneficiary, uint256 amount, uint256 cliffDays, uint256 durationDays) external onlyOwner {
        require(vestings[beneficiary].totalAmount == 0, "Already vested");

        VestingSchedule memory schedule = VestingSchedule({
            totalAmount: amount,
            released: 0,
            startTime: block.timestamp,
            cliffTime: block.timestamp + cliffDays * 1 days,
            duration: durationDays * 1 days
        });

        vestings[beneficiary] = schedule;
        totalVested += amount;
        token.transferFrom(msg.sender, address(this), amount);
    }

    function release() external {
        VestingSchedule storage schedule = vestings[msg.sender];
        require(schedule.totalAmount > 0, "No vesting");

        uint256 vested = _vestedAmount(schedule);
        uint256 releasable = vested - schedule.released;

        require(releasable > 0, "Nothing to release");

        schedule.released += releasable;
        token.transfer(msg.sender, releasable);
    }

    function _vestedAmount(VestingSchedule memory schedule) internal view returns (uint256) {
        if (block.timestamp < schedule.cliffTime) return 0;
        if (block.timestamp >= schedule.startTime + schedule.duration) return schedule.totalAmount;

        uint256 timePassed = block.timestamp - schedule.cliffTime;
        return (schedule.totalAmount * timePassed) / schedule.duration;
    }
}
