// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract RentalContract {
    // 定义房源信息的结构体
    struct RentalProperty {
        string addr;
        uint256 area;
        uint256 rent;
        address payable landlord;
        address tenant;
        bool isRented;
        bytes32 hashLock;
        uint256 timeLock;
        uint256 nextRentDue; // 下一次租金到期时间
        bool badReputation; // 房东是否有恶意行为
    }

    // 映射房源ID到房源信息
    mapping(uint256 => RentalProperty) public properties;
    // 房源ID计数器
    uint256 public propertyCount;
    uint256 public timeLock = 30 days;
    uint256 public book_fee = 100 wei;
    uint256 public recovery_fee = 100 wei;
    uint256 public publish_fee = 100 wei;

    // 发布房源信息的事件
    event PropertyPublished(uint256 propertyId, string addr, uint256 area, uint256 rent);
    // 租房合约创建的事件
    event RentingContractCreated(uint256 propertyId, address tenant, uint256 rent, uint256 timeLock);
    // 合约解锁的事件
    event ContractUnlocked(uint256 propertyId, address landlord);
    // 合约退款的事件
    event ContractRefunded(uint256 propertyId, address tenant);
    // 续租的事件
    event LeaseRenewed(uint256 propertyId, address tenant, uint256 rent, uint256 nextDueDate);
    // 租约到期，押金返还给租客
    event DepositReturned(uint256 propertyId, address tenant, uint256 deposit);

    // 发布房源信息的功能函数
    function publishProperty(string memory _addr, uint256 _area, uint256 _rent) public payable{
        require(msg.value == publish_fee, "Incorrect publish fee amount");
        require(_rent > 0, "Rent must be greater than zero");
        propertyCount++;
        properties[propertyCount] = RentalProperty({
            addr: _addr,
            area: _area,
            rent: _rent,
            landlord: payable(msg.sender),
            tenant: address(0),
            isRented: false,
            hashLock: 0x0,
            timeLock: 0,
            nextRentDue: 0,
            badReputation : false
        });
        emit PropertyPublished(propertyCount, _addr, _area, _rent);
    }

    // 房客支付押金和租金的功能函数
    function rentProperty(uint256 _propertyId, bytes32 _hashLock ) public payable {
         // 时间锁定期限为30天
        RentalProperty storage property = properties[_propertyId];
        require(property.badReputation == false, "Property has been reported as bad reputation");
        require(!property.isRented, "Property is already rented");
        require(msg.value == property.rent + property.rent + recovery_fee + book_fee,"Incorrect payment amount "); // 押金，第一个月租金，维修费
        property.isRented = true;
        property.tenant = msg.sender;
        property.hashLock = _hashLock;
        property.timeLock = block.timestamp + timeLock;
        property.nextRentDue = block.timestamp + 30 days; // 假设按月续租
        emit RentingContractCreated(_propertyId, msg.sender, property.rent, timeLock);
    }

    // 房东解锁HTLC的功能函数
    function unlockHTLC(uint256 _propertyId, string memory _preimage) public {
        RentalProperty storage property = properties[_propertyId];
        require(property.hashLock != 0x0, "Hash lock must not be zero");
        require(property.isRented, "Property is not rented");
        require(msg.sender == property.landlord, "Only landlord can unlock");
        require(sha256(bytes(_preimage)) == property.hashLock, "Invalid preimage");
        
        // 将订金和第一个月的租金转给房东
        property.landlord.transfer(book_fee +property.rent);
        property.hashLock = 0x0; // 重置哈希锁
        property.timeLock = 0; // 重置时间锁
        emit ContractUnlocked(_propertyId, msg.sender);
    }

    // 时间锁到期自动退款的功能函数
    function refundHTLC(uint256 _propertyId) public {
        RentalProperty storage property = properties[_propertyId];
        require(property.isRented, "Property is not rented");
        require(block.timestamp >= property.timeLock, "Time lock has not expired");
        require(address(this).balance >= property.rent + property.rent  + recovery_fee/2 , "Contract balance is insufficient");

        // 将押金和租金,维修费的一半退还给房客

        payable(property.tenant).transfer(property.rent*2+recovery_fee/2 );
        property.isRented = false; // 重置租赁状态
        property.hashLock = 0x0; // 重置哈希锁
        property.timeLock = 0; // 重置时间锁
        property.nextRentDue = 0; // 重置下一次租金到期时间
        property.tenant = address(0); // 重置租客地址
        property.badReputation = true; // 房东有恶意行为
        emit ContractRefunded(_propertyId, property.tenant);
    }

    // 续租的功能函数
    function renewLease(uint256 _propertyId) public payable {
        RentalProperty storage property = properties[_propertyId];
        require(property.isRented, "Property is not rented");
        require(msg.sender == property.tenant, "Only tenant can renew the lease");
        require(msg.value == property.rent, "your given value not equal to the rent");

        // 更新下一次租金到期时间
        property.nextRentDue = property.nextRentDue + 30 days; // 假设租期为每月一次
        // 将租金支付给房东
        property.landlord.transfer(msg.value);
        emit LeaseRenewed(_propertyId, msg.sender, msg.value, property.nextRentDue);
    }

    // 查询房源信息的功能函数
    function getProperty(uint256 _propertyId) public view returns (RentalProperty memory) {
        return properties[_propertyId];
    }

    // 租约到期，返还押金
    function returnDeposit(uint256 _propertyId) public {
        RentalProperty storage property = properties[_propertyId];
        require(property.isRented, "Property is not rented");
        require(msg.sender == property.landlord, "Only landlord can return deposit");
        require(block.timestamp >= property.nextRentDue, "Lease period has not ended");
        uint256 deposit = property.rent; // 押金为一个月房租
        payable(property.tenant).transfer(deposit);
        emit DepositReturned(_propertyId, property.tenant, deposit);
         // 将房产状态更新为未出租
        property.isRented = false;
        property.tenant = address(0); // 重置租客地址
        property.hashLock = 0x0; // 重置哈希锁
        property.timeLock = 0; // 重置时间锁
        property.nextRentDue = 0; // 重置下一次租金到期时间
    }

}
