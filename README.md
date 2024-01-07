# cannacoinbdev
Cannacoin is an developing cryptocurrency project that aims to provide a store of value as well as a medium of exchange for cannabis consumers and businneses in areas where its legal. We are seeking any and all collaboration on this project and welcome any feedback on how we can improve the contract before it is deployed on a main net. We are looking to launch on the Cronos network if possible. 

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Cannacoin is ERC20Burnable, Ownable, ReentrancyGuard {
    using SafeMath for uint256;

    IERC20 public token;

    // Staking and vault balances
    mapping(address => uint256) public balanceOf;
    mapping(address => uint256) public stakedBalance;
    mapping(address => mapping(uint256 => uint256)) public vaultRewards;

    // Struct for vault details
    struct Vault {
        uint256 amount;
        uint256 releaseTime;
    }

    // Mapping for user vaults
    mapping(address => Vault[]) public vaults;

    // Events for staking, unstaking, vault operations, and reward claiming
    event Staked(address indexed staker, uint256 amount);
    event Unstaked(address indexed staker, uint256 amount);
    event VaultLocked(address indexed locker, uint256 amount, uint256 releaseTime, uint256 timestamp);
    event VaultUnlocked(address indexed locker, uint256 amount, uint256 timestamp);
    event RewardClaimed(address indexed staker, uint256 amount, uint256 timestamp);

    // Penalty parameters
    uint256 public basePenaltyPercentage = 3; // Base penalty on each transaction
    uint256 public additionalPenaltyPercentage = 2; // Additional penalty for large transactions
    uint256 public largeTransactionThreshold = 750000 * 10**18; // Threshold for considering a transaction as large
    uint256 public largeTransactionPenaltyPercentage = 7; // Penalty percentage for large transactions

    // Maximum allowed balance
    uint256 public maxBalance = 25000000 * 10**18; // Maximum allowed balance

    // Governance parameters
    uint256 public minCoinsForGovernance = 500; // Minimum coins required for governance
    mapping(address => bool) public isAdmin; // Mapping of admin status for each address
    mapping(address => bool) public isMember; // Mapping of membership status for each address
    mapping(address => uint256) public votePower; // Mapping of vote power for each address

    // Governance events
    event AdminAdded(address indexed admin);
    event AdminRemoved(address indexed admin);
    event MemberAdded(address indexed member);
    event MemberRemoved(address indexed member);
    event VotePowerUpdated(address indexed member, uint256 votePower);

    // Voting parameters
    uint256 public votingDuration = 14 days; // Duration for which a vote is open
    uint256 public proposalPassPercentage = 60; // Percentage needed for a proposal to pass
    mapping(address => mapping(uint256 => bool)) public hasVoted; // Mapping to track whether an address has voted on a proposal
    mapping(address => uint256) public voteLockExpiration; // Mapping to track when an address's vote lock expires

    // Voting events
    event VoteCasted(address indexed voter, uint256 proposalIndex, bool support);

    // Vault lock periods and APRs
    uint256[] public lockPeriods = [0, 30 days, 90 days, 180 days, 365 days]; // Lock periods in seconds
    uint256[] public aprs = [0, 10, 36, 75, 200]; // Corresponding APRs in percentage

    // Event for penalty burning
    event PenaltyBurn(address indexed sender, address indexed recipient, uint256 amount, uint256 penaltyAmount);

    // Modifier for only members
    modifier onlyMember() {
        require(isMember[msg.sender], "Not a member");
        _;
    }

    // Modifier for only admins
    modifier onlyAdmin() {
        require(isAdmin[msg.sender], "Not an admin");
        _;
    }

    // Modifier for only members and admins
    modifier onlyMemberOrAdmin() {
        require(isMember[msg.sender] || isAdmin[msg.sender], "Not a member or admin");
        _;
    }

    // Modifier to prevent reentrancy
    modifier nonReentrant() {
        require(!inLock, "Reentrant call");
        inLock = true;
        _;
        inLock = false;
    }

    // Modifier to check if the address is valid
    modifier validAddress(address _addr) {
        require(_addr != address(0), "Invalid address");
        _;
    }

    // Modifier to check if the address has a balance
    modifier hasBalance(address _addr, uint256 _amount) {
        require(balanceOf[_addr] >= _amount, "Insufficient balance");
        _;
    }

    // Modifier to check gas limit
    modifier checkGasLimit() {
        require(gasleft() >= gasLimit, 1000000


