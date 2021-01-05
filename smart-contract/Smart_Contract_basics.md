---
layout: post
lang: en
lang-ref: Smart_Contract_basics
---

# NEO Smart contract 101

>
> **Objective**: Learn smart contract basic elements
>
> **Main points**:
>
> 1. The Strucutre of Smart contract
>
> 2. The Storage usage in Smart contract
>
> 3. Triggers and CheckWitness
>
> 4. Some contract Demos
>  

<p align="center">
  <img width="60%" src="./imgs/newbie.jpg">
</p>
This guide walks you through the process of creating a basic smart contract with NEO smart contract framework.

Let's have a look at our basic `hello world` contract.

```C#
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;
using System;

namespace Helloworld
{
    public class Contract : SmartContract
    {
        private const string test_str = "Hello World";
        public static String Hello()
        {
            Storage.Put("Hello", "World");
            return test_str;
        }
    }
}
```

This contract is concise and simple, but there is plenty going on under the hood. We break it down step by step.
##  Contract structure

Each smart contract inherits the `SmartContract` base class which is defined in the NEO framework and provides some basic methods.

In the `NEO` namespace there are rich APIs provided by the Neo blockchain, providing a way to access the blockchain data and manipulate the persistent storage. These APIs can be divided into two categories:

1.  Blockchain ledger. The contract can access all the data on the entire blockchain through `interops` layer, including complete blocks and transactions, as well as each of their fields.

2.  Persistent storage. Each deployed contract on NEO can have a private storage space. With these APIs developers are capable to access the data in the contract.

## Contract fields
In the contract class, fields declared with the keyword `static readonly` or `const` will always contain the same values. For instance, we can define the contract's Owner or the factor used in the asset transfer as below:

```c#
// Define the owner of this contract with a fixed address, which usually is the contract creator.
public static readonly UInt60 Owner = "ATrzHaicmhRj15C3Vv6e6gLfLqhSD2PtTr".ToScriptHash();

// A constant value
private const ulong factor = 100000000;
```

we can always define these types of fields as contants in the contract to ensure their immutability throughout the execution.

In addition, developers can return a constant in a static method to expose the value. For example, you may need to name your newly created token, to achieve this, you can provide such a method so that users can know the name each time they call it.

```c#
public static string Name() =>  "name of the token";
```
## Storage property

When you develop the smart contract, you are allowed to store your application data on the blockchain. Each NEO full node keeps its own copy of the NEO blockchain and uses that copy to validate all transactions and blocks.

NEO has provided a key-value data store where the contract can access the data in its private storage with the specified key. Smart contracts can also send their storage contexts to other contracts, thereby entrusting other contracts to manage their storage areas. The NEO framework provides the `Storage` class for developers to read/write the persistent storage.  The `Storage` class is a static class so it does not contain an instance constructor. You can refer to [Storage API](https://docs.neo.org/en-us/sc/reference/fw/dotnet/neo/Storage.html) for more details.

For instance, you can store the total supply of your token like this:

```csharp
// Key is totalSupply and value is 100000000
Storage.Put(Storage.CurrentContext, "totalSupply", 100000000);
```

Here `CurrentContext` returns the current store context. The contract can pass the store context to other contracts (as a way of authorization), allowing other contracts to perform read/write operations on its storage.

You can just use `Storage` to store primitive values, while for structured data you should use `StorageMap` instead, which allows you to store the entire container in a single key.

```csharp
//Get the totalSupply in the storageMap. The Map is used an entire container with a key name "contract"
StorageMap contract = Storage.CurrentContext.CreateMap(nameof(contract));
return contract.Get("totalSupply").AsBigInteger();
```
## Data type
When using C# to develop smart contracts, you cannot use the full set of C# features due to the difference between NeoVM and Dotnet IL.

Because NeoVM is more compact, we can only compile limited C#/dotnet features into an NEF file.

NeoVM provides the following types：

-   `Array`
-   `Boolean`
-   `Buffer`
-   `ByteString`
-   `Integer`
-   `InteropInterface`
-   `Map`
-   `Null`
-   `Pointer`
-   `Struct`

The basic types of C# are:

-   `Int8 int16 int32 int64 uint8 uint16 uint32 uint64`
-   `float double`
-   `Boolean`
-   `Char String`

## Your first NEO contract

Now let us move on to write your first real-world smart contract. Here we provide a simple DNS system written in C#. Its main function is to store the domain name for users. It covers all the points we mentioned above. The source code is here:

```csharp
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;
using Neo;

namespace Example
{
    public class Domain : SmartContract
    {
        public static object Main(string operation, params object[] args)
        {
            if (Runtime.Trigger == TriggerType.Application)
            {
                switch (operation)
                {
                    case "query":
                        return Query((string)args[0]);
                    case "register":
                        return Register((string)args[0], (UInt160)args[1]);
                    case "delete":
                        return Delete((string)args[0], (UInt160)args[1]);
                    default:
                        return false;
                }
            }
            return false;
        }

        private static byte[] Query(string domain)
        {
            return Storage.Get(Storage.CurrentContext, domain);
        }

        private static bool Register(string domain, UInt160 owner)
        {
            // Check if the owner is the same as the one who invoke the contract
            if (!Runtime.CheckWitness(owner)) return false;
            byte[] value = Storage.Get(Storage.CurrentContext, domain);
            if (value != null) return false;
            Storage.Put(Storage.CurrentContext, domain, (byte[])owner);
            return true;
        }

        private static bool Delete(string domain, UInt160 owner)
        {
            if (!Runtime.CheckWitness(owner)) return false;
            Storage.Delete(Storage.CurrentContext, domain);
            return true;
        }
    }
}
```

##  Entry method

Here the `Main` method acts as the entry method, you can pass in the method name and the corresponsding parameters for desired invocation. In fact, any method in the smart contract declared with `public static` can be invoked as entry methods. Here is the example:
```c#
using Neo.SmartContract.Framework;

namespace Example
{
    class EntryMethod: SmartContract
    {
        public static object First()
        {
            return 'a';
        }
        public static object Second()
        {
            return 'b';
        }
    }
}
```

If you check the manifest file of this contract, you will find the `offset` attribute of the two methods in the `abi` part. Every time you invoke the contract with the specified method name, the corresponding offset will be assigned to the initialPosition so the execution engine can find the right method for furture execution.

### Trigger
A smart contract trigger is a mechanism that triggers the execution of smart contracts. There are six triggers introduced in the NEO smart contract，the most used are `Verification` and  `Application`.

### Verification trigger

When the contract address is included in the transaction signers, the `Verify` method defined in the contract will be called with the Verification trigger type to verify if the signature is correct. For example, this method needs to be called when withdrawing token from the contract.

```c#
public static bool Verify()
{
    return Runtime.CheckWitness(Owner);
}
```
### CheckWitness

It is likely that you need to check the address invoking the contract. The `Runtime.CheckWitness` method accepts a single parameter which represents the address that you would like to validate against the address used to invoke the contract code. In more deeper detail, it verifies that the transactions / block of the calling contract has validated the required script hashes.

Inside our `DNS smart contract`, the `Register` function firstly check if the owner is the same as the one who is invoking the contract. Here we use the `Runtime.CheckWitness` function. Then we try to fetch the domain owner first to see if the domain is already exists in the storage. If not, we can store our `domain -> owner` pair using the `Storage.Put` method.

```csharp
private static bool Register(string domain, UInt160 owner)
{
    // Check if  the owner is the same as the one who invoke the contract
    if (!Runtime.CheckWitness(owner)) return false;
    byte[] value = Storage.Get(Storage.CurrentContext, domain);
    if (value != null) return false;
    Storage.Put(Storage.CurrentContext, domain, (byte[])owner);
    return true;
}
```

Similarly, the `Delete` function checks the owner first and then deletes the pair using the `Storage.Delete` method. 

### Events

In Smart contract, events are a way to communicate something happening on the blockchain to your app front-end (or back-end), which can be used to `listen` for certain events and take action the moment they happen. You might use this function to update an external database, do analytics, or update a UI. In most cases the use of events is optional for you. While in NEP-17 Token standard, the event `Transfer` must be triggered when tokens are transferred, including zero value transfers and self-transfers.

```c#
//Should be called when caller transfer nep-17 asset.
[DisplayName("Transfer")]
public static event Action<byte[], byte[], BigInteger> OnTransfer;
```

Here `Transfer` is the event name.

### Json serialization

The Json serialization/deserialization feature is added in Neo3 framework:

```c#
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;

namespace Example
{
    public class Json : SmartContract
    {
        public static string Serialize(object obj)
        {
            return Json.Serialize(obj);
        }

        public static object Deserialize(string json)
        {
            return Json.Deserialize(json);
        }
    }
}
```
### Callback support

```c#
using System;
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;

public class Callback : SmartContract.Framework.SmartContract
{
    public static object CreateCallback()
    {
        var callback = Callback.Create(new Func<int>(test));
        return callback.Invoke(new object[0]);
    }

    public static int test()
    {
        return 1024;
    }
}
```
Here we use the `Callback.Create` method to define a callback instance to delegate the `test` method, and then invoke the callback method which will return a integer.