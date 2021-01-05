# NEP-17

The NEP-17 proposal is a replacement of the original NEP5 proposal, which outlines a token standard for the Neo blockchain that will provide systems with a generalized interaction mechanism for tokenized Smart Contracts. 

Operations on NEP17 assets are mainly involved in updating account balance in the storage area. Here we provide an example for a valid NEP-17 token definition:

```c#
using Neo;
using System;
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;
using Neo.SmartContract.Framework.Services.System;
using System.Numerics;
using System.ComponentModel;

namespace Example
{
    [SupportedStandards("NEP17", "NEP10")]
    public partial class NEP17 : SmartContract
    {
        private static readonly ulong InitialSupply = 2_000_000_000_000_000;
        private static readonly UInt160 Owner = "NXsG3zwpwcfvBiA3bNMx6mWZGEro9ZqTqM".ToScriptHash();

        private static StorageMap contract => Storage.CurrentContext.CreateMap(nameof(contract));
        private static StorageMap asset => Storage.CurrentContext.CreateMap(nameof(asset));
        
        [DisplayName("Transfer")]
        public static event Action<UInt160, UInt160, BigInteger> OnTransfer;
        
        public static string Symbol() => "MyToken";

        public static ulong Decimals() => 8;

        public static BigInteger TotalSupply() => contract.Get("totalSupply").ToBigInteger();

        public static BigInteger BalanceOf(UInt160 account)
        {
            byte[] accountArr = (byte[])account;
            if (accountArr.Length != 20) throw new Exception("The parameters account SHOULD be a 20-byte non-zero address.");
            return asset.Get(accountArr).ToBigInteger(); 
        }

        public static bool Transfer(UInt160 from, UInt160 to, BigInteger amount, object data)
        {
            byte[] fromArr = (byte[])from;
            byte[] toArr = (byte[])to;
            if (fromArr.Length != 20 || toArr.Length != 20) throw new Exception("The parameters from and to SHOULD be 20-byte non-zero addresses.");
            if (amount <= 0) throw new Exception("The parameter amount MUST be greater than 0.");
            if (!Runtime.CheckWitness(from) && !from.Equals(ExecutionEngine.CallingScriptHash)) throw new Exception("No authorization.");
            BigInteger balance = asset.Get((byte[])from).ToBigInteger();
            if (balance < amount) throw new Exception("Insufficient balance.");
            if (from == to) return true;

            if (balance == amount)
                asset.Delete(fromArr);
            else
                asset.Put(fromArr, balance - amount);

            var toAmount = asset.Get(toArr).ToBigInteger();
            asset.Put(toArr, toAmount + amount);

            OnTransfer(from, to, amount);

            // Validate payable
            if (ManagementContract.GetContract(to) != null) Contract.Call(to, "onPayment", new object[] { from, amount, data });
            return true;
        }

        public static void _deploy(bool update)
        {
            if(update) return;
            if (contract.Get("totalSupply").ToBigInteger() > 0) throw new Exception("Contract has been deployed.");
            contract.Put("totalSupply", InitialSupply);
            asset.Put((byte[])Owner, InitialSupply);

            OnTransfer(null, Owner, InitialSupply);
        }
    }
}
```

First of all, we compile the contract to generate the `nef` and `manifest` files. Then we can deploy the contract in neo-cli by typing the command `deploy nep17.nef`, which will automatically call the `_deploy` method to initiate the supply of the token and update the corresponding storage area. In order to define a valid NEP-17 token, we must provide the following methods:

**totalSupply**    

```c#
public static BigInteger totalSupply()
```

Returns the total token supply deployed in the system.

**symbol**

```c#
public static string symbol()
```

Returns a short string symbol of the token managed in this contract. e.g. "MYT". 

This symbol must be a valid ASCII string, with no whitespace characters or new-lines, and should be short (3-8 characters is recommended) uppercase latin alphabet (i.e. the 26 letters used in English).

This method MUST always return the same value every time it is invoked.

**decimals**

```c#
public static byte decimals()
```

Returns the number of decimals used by the token - e.g. 8, means to divide the token amount by 100,000,000 to get its user representation.

This method MUST always return the same value every time it is invoked.

**balanceOf**

```c#
public static BigInteger balanceOf(byte[] account)
```

Returns the token balance of the `account`.

The parameter `account` must be a 20-byte address. If not, this method should throw an exception.

If the `account` is an unused address, this method must return 0.

**transfer**

```c#
public static bool transfer(byte[] from, byte[] to, BigInteger amount)
```

The parameters `from` and `to` should be 20-byte addresses. If not, this method should throw an exception.<br/>

The parameter amount must be greater than or equal to 0. If not, this method should throw an exception.<br/>

The function must return false if the `from` account balance does not have enough tokens to spend.<br/>

If the method succeeds, it must fire the `Transfer` event, and must return `true`, even if the `amount` is 0, or `from` and `to` are the same address.<br/>

The function SHOULD check whether the `from` address equals the caller contract hash. If so, the transfer SHOULD be processed; If not, the function SHOULD use the SYSCALL `Neo.Runtime.CheckWitness` to verify the transfer.<br/>

If the transfer is not processed successfully, the function must return `false`.

If `to` is the address hash of a deployed contract, the function must invoke the `onPayment` method of the contract after triggering the `Transfer` event. If the contract `to` does not want to receive the transfer, it must invoke the opcode `ABORT`.

Assuming that we enable the payment function of the contract by calling the `EnablePayment` method, every time we transfer `NEO` or `GAS` to this contract, the `OnPayment` method will be invoked to mint some tokens for the sender of the transaction.

```c#
using Neo;
using System;
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;
using Neo.SmartContract.Framework.Services.System;
using System.Numerics;
using System.ComponentModel;

namespace Example
{
    public partial class NEP17 : SmartContract
    {
        static readonly ulong TokensPerNEO = 1_000_000_000;
        static readonly ulong TokensPerGAS = 1;
        static readonly ulong MaxSupply = 10_000_000_000_000_000;

        public static void OnPayment(UInt160 from, BigInteger amount, object data)
        {
            if (asset.Get("enable").ToBigInteger().Equals(1))
            {
                if (ExecutionEngine.CallingScriptHash == NEO.Hash)
                {
                    Mint(amount * TokensPerNEO);
                }
                else if (ExecutionEngine.CallingScriptHash == GAS.Hash)
                {
                    if (from != null) Mint(amount * TokensPerGAS);
                }
                else
                {
                    throw new Exception("Wrong calling script hash");
                }
            }
            else
            {
                throw new Exception("Payment is disable on this contract!");
            }

        }

        private static void Mint(BigInteger amount)
        {
            var totalSupply = contract.Get("totalSupply").ToBigInteger(); ;
            if (totalSupply <= 0) throw new Exception("Contract not deployed.");

            var avaliable_supply = MaxSupply - totalSupply;

            if (amount <= 0) throw new Exception("Amount cannot be zero.");
            if (amount > avaliable_supply) throw new Exception("Insufficient supply for mint tokens.");

            Transaction tx = (Transaction)ExecutionEngine.ScriptContainer;
            BigInteger balance = asset.Get((byte[])tx.Sender).ToBigInteger();
            asset.Put((byte[])tx.Sender, balance + amount);
            contract.Put("totalSupply", totalSupply + amount);

            OnTransfer(null, tx.Sender, amount);
        }

        public static void EnablePayment()
        {
            if (!Runtime.CheckWitness(Owner)) throw new Exception("No authorization.");
            asset.Put("enable".ToByteArray(), 1);
        }

        public static void DisablePayment()
        {
            if (!Runtime.CheckWitness(Owner)) throw new Exception("No authorization.");
            asset.Put("enable".ToByteArray(), 0);
        }
    }
}
``` 
**Transfer**
```c#
[DisplayName("Transfer")]
public static event Action<UInt160, UInt160, BigInteger> OnTransfer;
```
The event `Transfer` must be triggered when tokens are transferred, including zero value transfers and self-transfers.

A token contract which creates new tokens must trigger a `Transfer` event with the from address set to null when tokens are created.

A token contract which burns tokens must trigger a `Transfer` event with the to address set to null when tokens are burned.




