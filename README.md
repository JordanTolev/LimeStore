// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;
//Everything is in one file, because i had trouble with the importing system in remix
//Is There a template functionality in Solidity for Libraries
library IterableMappingProduct {

   struct ProductItem{
        address productAddress;
        address productOwner;
        uint productPrice;
        string productName;
        uint productId;
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

library IterableMappingTransaction {

   struct Transaction{
        uint ProductId;
        address ProductAddress;
        uint TimeOfTransaction;
    }

    struct TransactionMap {
        uint productsInTransactionsCount;
        address [] productAdresses;
        mapping(address => uint) transactionsPerClient;
        mapping(address => Transaction[]) values;
        
    }

    function get(TransactionMap storage map, address key) public view returns (Transaction [] storage) {
        return map.values[key];
    }

    function getClientTransactions(TransactionMap storage map) public view returns (mapping(address => Transaction[]) storage) {
        return map.values;
    }

    function hasClientBoughtProduct(TransactionMap storage map, address clientAddr, address productAddr) public view returns (bool){
        Transaction [] storage clientTransactions = map.values[clientAddr];
        for(uint i = 0; i < map.transactionsPerClient[clientAddr]; i++){
            if(clientTransactions[i].ProductAddress == productAddr){
                return true;
            }
        }
        return false;
    }

    function doesProductExist(TransactionMap storage map, address productAddr) public view returns (bool){
        for(uint i = 0; i < map.productsInTransactionsCount; i++){
            if(map.productAdresses[i] == productAddr){
                return true;
            }
        }
        return false;
    }


    function getProductAdresses(TransactionMap storage map) public view returns (address [] storage){
        return map.productAdresses;
    }

    function sizeOfProductAddresses(TransactionMap storage map) public view returns (uint) {
        return map.productsInTransactionsCount;
    }

    function set(TransactionMap storage map, address key, Transaction memory val) public {
        map.productsInTransactionsCount++;
        map.productAdresses.push(val.ProductAddress);
        map.values[key].push(val);
        map.transactionsPerClient[key]++;
    }

    function remove(TransactionMap storage map, address key, address productAddr) public {
        deleteFromProducts(map, productAddr);
        map.productsInTransactionsCount--;
        deleteFromTransactions(map, key, productAddr);
        map.transactionsPerClient[key]--;

    }

    function deleteFromProducts(TransactionMap storage map, address productAddr) public {
        uint indexToBeDeleted = getIndexOfProduct(map, productAddr);
        if(indexToBeDeleted != 0){
            map.productAdresses[indexToBeDeleted] = map.productAdresses[ map.productsInTransactionsCount - 1];
            map.productAdresses.pop();
        }
    }

    function getIndexOfProduct(TransactionMap storage map, address productAddr) private view returns(uint){
        for(uint i = 0; i < map.productsInTransactionsCount; i++){
            if(map.productAdresses[i] == productAddr){
                return i;
            }
        }
        return 0;
    }

    function deleteFromTransactions(TransactionMap storage map,  address key, address productAddr) public {
        uint indexToBeDeleted = getIndexOfTransaction(map, key, productAddr);
        if(indexToBeDeleted != 0){
            map.values[key][indexToBeDeleted].ProductId =  map.values[key][map.transactionsPerClient[key] - 1].ProductId;
            map.values[key][indexToBeDeleted].ProductAddress =  map.values[key][map.transactionsPerClient[key] - 1].ProductAddress;
            map.values[key][indexToBeDeleted].TimeOfTransaction =  map.values[key][map.transactionsPerClient[key] - 1].TimeOfTransaction;

            map.values[key].pop();
        }
    }

    function getIndexOfTransaction(TransactionMap storage map, address key, address productAddr) private view returns(uint){
        Transaction [] storage clientTransactions = map.values[key];
        for(uint i = 0; i < map.transactionsPerClient[key]; i++){
            if(clientTransactions[i].ProductAddress == productAddr){
                return i;
            }
        }
        return 0;
    }
}


contract Ownable {
    address internal owner;
    
    modifier onlyOwner() {
        require(owner == msg.sender, "Must be owner");
        _;
    }
    
    constructor() {
        owner = payable(msg.sender);
    }
}

contract Product {
    address public productOwner;
    string public name;
    uint public price;
    uint public productId;
    mapping(address => uint) private previousOwners;

    modifier isOwner() {
        require(productOwner == msg.sender, "Must be owner for this action");
        _;
    }

    modifier eligibleForRefund(address prOwner) {
        require(previousOwners[prOwner] != 0, "Not a previous owner");
        //require is needed for the functionality of a refund in a certain period of time
        // require(previousOwners[prOwner] + 10000 <=  block.timestamp, "No longer owner");
        _;
    }

    constructor(address _productOwner, string memory _name, uint _price, uint _product_id){
        productOwner = _productOwner;
        name = _name;
        price = _price;
        productId = _product_id;
    }

    function addPreviousOwner(address prOwner, uint timeOfTransaction) public isOwner{
        previousOwners[prOwner] = timeOfTransaction;
    }

    function transferTo(address clientAddr) public isOwner{
        productOwner = clientAddr;
    }

    function safeTransfer(address previousOwner) public eligibleForRefund(previousOwner) {
        productOwner = previousOwner;
    }
}

contract LimeStore is Ownable{
    using IterableMappingProduct for IterableMappingProduct.Map;
    using IterableMappingTransaction for IterableMappingTransaction.TransactionMap;
    
    event LogReceivePayment(address client, uint value);
    event LogNewOwner(address owner);
    event LogAddedProduct(address productAddr, address owner, string name, uint id);
    event LogBuyTransaction(address newProductOwner, uint moneySend, uint refund, uint productId, uint productPrice, address prAddr);
    event LogRefund(address clientAddr, string productName, uint productPrice);


    struct TransactionInfo{
        uint productId;
        address productAddress;
        uint timeOfTransaction;
    }

    uint private PID;
    IterableMappingProduct.Map private productItems;
    mapping(string => uint) public productQuantities;
    IterableMappingTransaction.TransactionMap public transactionHistory;
    mapping(address => address[]) public productOwnerHistory;

    constructor() Ownable(){

    }

    fallback() external payable {
       emit LogReceivePayment(msg.sender, msg.value);
    }

    function changeOwner(address _newOwner) external onlyOwner{
        require(_newOwner != address(0), "invalid address");
        owner = _newOwner;
        emit LogNewOwner(_newOwner);
    }

    function addProduct(IterableMappingProduct.ProductItem memory productItem) public onlyOwner{
        bytes memory tempEmptyStringTest = bytes(productItem.productName);
        require(tempEmptyStringTest.length > 0, "invalid name for product");
        require(productItem.productPrice > 0, "invalid product price");


        if(productExistsInStorageByAddress(productItem.productAddress) || transactionHistory.doesProductExist(productItem.productAddress)){
            revert("Cannot add an item that already exists");
        }
        else{
            Product productToAdd = new Product(address(this), productItem.productName, productItem.productPrice, PID);
            productItem.productAddress = address(productToAdd);
        }

        PID++;
        productItem.productOwner = address(this);
        productItem.productId = PID;
        productItems.set(PID, productItem);
        productQuantities[productItem.productName]++;
        productOwnerHistory[productItem.productAddress].push(owner);
        emit LogAddedProduct(productItem.productAddress, productItem.productOwner, productItem.productName, PID);
    }
    
    function buyProduct(uint prId) external payable{
        require(prId > 0, "Id should be a number > 0");
        require(productExistsInStorageById(prId), "No item in storage");

        IterableMappingProduct.ProductItem memory productToSell = productItems.get(prId);
        uint productPrice = productToSell.productPrice;
        uint paymentAmount = uint(msg.value);

        require(productPrice <= paymentAmount, "Insufficient funds");
        require(!transactionHistory.hasClientBoughtProduct(msg.sender, productToSell.productAddress), "client cannot buy the same product twice");

        Product(productToSell.productAddress).addPreviousOwner(address(this), block.timestamp);
        (bool successTransfer, ) = productToSell.productAddress.call(
            abi.encodeWithSignature("transferTo(address)", "call transferTo", msg.sender)
        );
        require(successTransfer, "Product transfer unsuccesfull");

        productQuantities[productToSell.productName]--;
        productItems.remove(prId);

        uint refund = paymentAmount - productPrice;
        if(refund > 0){
            payable(msg.sender).transfer(refund);
        }

        IterableMappingTransaction.Transaction memory trInfo =  IterableMappingTransaction.Transaction(prId, productToSell.productAddress, block.timestamp);
        transactionHistory.set(msg.sender, trInfo);
        productOwnerHistory[productToSell.productAddress].push(msg.sender);
        emit LogBuyTransaction(msg.sender, msg.value, refund, prId, productPrice, productToSell.productAddress);
    }

    function productExistsInStorageById(uint ID) private view returns(bool){
        for(uint i = 0; i < productItems.size(); i++){
            if(productItems.getValueAtIndex(i).productId == ID){
                return true;
            }
        }
        return false;
    }

    function productExistsInStorageByAddress(address prAddr) private view returns(bool){
        for(uint i = 0; i < productItems.size(); i++){
            if(productItems.getValueAtIndex(i).productAddress == prAddr){
                return true;
            }
        }
        return false;
    }

    function refundProduct(IterableMappingProduct.ProductItem memory productTorefund) external payable{
        require(transactionHistory.hasClientBoughtProduct(msg.sender, productTorefund.productAddress), "cannot refund something you have not bought");
        require(address(this).balance >= productTorefund.productPrice, "Store doesnt have enough fund for refund");

        //I know this gives the store the power to get its items back in the refund time, but i could not think of a better solution for refunding
        Product(productTorefund.productAddress).safeTransfer(address(this));
        payable(msg.sender).transfer(productTorefund.productPrice);

        transactionHistory.remove(msg.sender, productTorefund.productAddress);
        
        PID++;
        productTorefund.productOwner = address(this);
        productTorefund.productId = PID;
        productItems.set(PID, productTorefund);
        productQuantities[productTorefund.productName]++;
        emit LogRefund(msg.sender, productTorefund.productName, productTorefund.productPrice);
    }

    function withdrawFunds(uint amount) external onlyOwner {
        require(address(this).balance >= amount, "Not enough funds");
        payable(msg.sender).transfer(amount);
    }

}
