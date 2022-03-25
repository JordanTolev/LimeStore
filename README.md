// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

library IterableMapping {

   struct ProductItem{
        address productAddress;
        address productOwner;
        uint productPrice;
        string productName;
    }

    struct Map {
        uint[] keys;
        mapping(uint => ProductItem) values;
        mapping(uint => bool) inserted;
        mapping(uint => uint) indexOf;
    }

    function get(Map storage map, uint key) public view returns (ProductItem storage) {
        return map.values[key];
    }

    function getKeyAtIndex(Map storage map, uint index) public view returns (uint) {
        return map.keys[index];
    }

    function size(Map storage map) public view returns (uint) {
        return map.keys.length;
    }

    function getValueAtIndex(Map storage map, uint index) public view returns (ProductItem storage) {
        return map.values[map.keys[index]];
    }

    function set(Map storage map, uint key, ProductItem memory val) public {
        if (map.inserted[key]) {
            map.values[key] = val;
        } else {
            map.inserted[key] = true;
            map.values[key] = val;
            map.indexOf[key] = map.keys.length;
            map.keys.push(key);
        }
    }

    function remove(Map storage map, uint key) public {
        if (!map.inserted[key]) {
            return;
        }

        delete map.inserted[key];
        delete map.values[key];

        uint index = map.indexOf[key];
        uint lastIndex = map.keys.length - 1;
        uint lastKey = map.keys[lastIndex];

        map.indexOf[lastKey] = index;
        delete map.indexOf[key];

        map.keys[index] = lastKey;
        map.keys.pop();
    }
}


contract Ownable {
    address internal owner;
    
    modifier onlyOwner() {
        require(owner == msg.sender, "Must be owner for this action");
        _;
    }
    
    constructor() {
        owner = msg.sender;
    }
}

contract Product {
    address public productOwner;
    string public name;
    uint public price;
    uint public productId;
    mapping(address => uint) private previousOwners;

    modifier onlyOwner() {
        require(productOwner == msg.sender, "Must be owner for this action");
        _;
    }

    modifier eligibleForRefund(address prOwner) {
        require(previousOwners[prOwner] != 0, "Not a previous owner");
        require(previousOwners[prOwner] + 100 <=  block.timestamp, "No longer owner");
        _;
    }

    constructor(address _productOwner, string memory _name, uint _price, uint _product_id){
        productOwner = _productOwner;
        name = _name;
        price = _price;
        productId = _product_id;
    }

    function addPreviousOwner(address prOwner, uint timeOfTransaction) public onlyOwner {
        previousOwners[prOwner] = timeOfTransaction;
    }

    function transferTo(address clientAddr) public onlyOwner {
        productOwner = clientAddr;
    }

    function safeTransfer(address previousOwner) public eligibleForRefund(previousOwner) {
        productOwner = previousOwner;
    }
}

contract LimeStore is Ownable{
    using IterableMapping for IterableMapping.Map;

    event LogReceivePayment(address client, uint value);
    event LogNewOwner(address owner);
    event LogAddedProduct(string name, uint id);
    event LogBuyTransaction(address newProductOwner, uint productId, uint productPrice);
    event LogRefund(address clientAddr, string productName, uint productPrice);


    struct TransactionInfo{
        uint productId;
        address productAddress;
        uint timeOfTransaction;
    }

    uint private PID;
    IterableMapping.Map private productItems;
    mapping(uint => uint) public productQuantities;
    mapping(address => TransactionInfo) public transactionHistory;

    constructor(){
        owner = payable(msg.sender);
    }

    fallback() external payable {
       emit LogReceivePayment(msg.sender, msg.value);
    }

    function changeOwner(address _newOwner) external onlyOwner{
        require(_newOwner != address(0), "invalid address");
        owner = _newOwner;
        emit LogNewOwner(_newOwner);
    }

    function addProduct(string memory _name, uint _price) public onlyOwner{
        bytes memory tempEmptyStringTest = bytes(_name);
        require(tempEmptyStringTest.length > 0, "invalid name for product");
        require(_price > 0, "invalid product price");

        Product productToAdd = new Product(owner, _name, _price, PID);
        IterableMapping.ProductItem memory productItem = IterableMapping.ProductItem(address(productToAdd), owner, _price, _name);
        PID++;

        if(productExistsInStorage(address(productToAdd))){
            revert("You cannot add the same item to the store");
        }
        else{
            productItems.set(PID, productItem);
            if(productQuantities[PID] != 0){
                productQuantities[PID] += 1;
            }
            else{
                productQuantities[PID] = 1;
            }

            emit LogAddedProduct(_name, PID);
        }

    }

    function productExistsInStorage(address prAddr) private view returns(bool flag){
        for(uint i = 0; i < productItems.size(); i++){
            if(productItems.getValueAtIndex(i).productAddress == prAddr){
                flag = true;
            }
        }
    }

    function buyProduct(uint prId) external payable{
        require(prId > 0, "Id should be a number > 0");
        require(productQuantities[prId] > 0, "No item in storage");

        IterableMapping.ProductItem memory productToSell = productItems.get(prId);
        uint productPrice = productToSell.productPrice;
        uint paymentAmount = uint(msg.value);

        require(productPrice <= paymentAmount, "Insufficient funds");
        require(transactionHistory[msg.sender].productAddress == address(0), "client cannot buy the same product twice");

        Product(productToSell.productAddress).addPreviousOwner(address(this), block.timestamp);
        Product(productToSell.productAddress).transferTo(msg.sender);

        if(productQuantities[prId] == 0){
            productItems.remove(prId);
        }
        else{
            productQuantities[prId] -= 1;
        }

        uint refund = paymentAmount - productPrice;
        if(refund > 0){
            payable(msg.sender).transfer(refund);
        }

        TransactionInfo memory trInfo = TransactionInfo(prId, productToSell.productAddress, block.timestamp);
        transactionHistory[msg.sender] = trInfo;

        emit LogBuyTransaction(msg.sender, prId, productPrice);
    }

    function refundProduct(IterableMapping.ProductItem memory productTorefund) external payable{
        require(transactionHistory[msg.sender].productAddress == productTorefund.productAddress, "cannot refund something you have not bought");
        require(address(this).balance >= productTorefund.productPrice, "Store doesnt have enough fund for refund");

        //I know this gives the store the power to get its items back in the refund time, but i could not think of a better solution for refunding
        Product(productTorefund.productAddress).safeTransfer(address(this));
        payable(msg.sender).transfer(productTorefund.productPrice);

        addProduct(productTorefund.productName, productTorefund.productPrice);
        emit LogRefund(msg.sender, productTorefund.productName, productTorefund.productPrice);
    }

    function withdrawFunds(uint amount) external onlyOwner {
        require(address(this).balance >= amount, "Not enough funds");
        payable(msg.sender).transfer(amount);
    }

}
