---
weight: 5
title: "2023 MetaTrust Web3 Security CTF Sui Challenges Writeup"
date: 2023-09-19T17:55:28+08:00
lastmod: 2023-09-19T17:55:28+08:00
draft: false
author: "Qi Qin"
authorLink: "https://leoq7.com"
description: "Writeup for 2023 MetaTrust Web3 Security CTF Sui Challenges"
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

Last week, I participated solo in the [2023 MetaTrust Web3 Security CTF](https://ctf.metatrust.io/). In addition to the traditional Solidity challenges, Mysten Labs and OtterSec provided four Sui Move challenges as a separate category. Fortunately, I managed to get two first bloods and two second bloods, placing me first in this specific track. Apart from these four challenges, there was also an intriguing challenge involving Move bytecode reverse engineering. Since I have previously written several related writeups, I won't delve into it here. Interested readers can refer to the excellent write-ups by [ashleyhsu.eth](https://ashirleyshe.github.io/p/metatrust-ctf-writeup-bytesmove/) and [jinu.eth](https://x.com/lj1nu/status/1702565727658742223?s=20).

## Challenge 1: Hello World

{{< admonition note "Challenge Info" >}}
Say hello to Douglas Adam.

- Host: metatrustctf.sui.io
- Port: 31337
- Source: https://github.com/MetaTrustLabs/ctf/tree/master/hellowWorld/
- Score: 10
{{< /admonition >}}

### Target contract

As the name suggests, the challenge 1 is a sanity-check to let players get familiar with Sui and the CTF framework.
```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/hellowWorld/framework/src/main.rs#L100-L114 */
// Check Solution
let mut args2: Vec<SuiValue> = Vec::new();
let arg_ob2 = SuiValue::Object(FakeID::Enumerated(1, 0));
args2.push(arg_ob2);

let ret_val = sui_ctf_framework::call_function(
    &mut adapter,
    chall_addr,
    chall,
    "is_owner",
    args2,
    Some("challenger".to_string()),
);
println!("[SERVER] Return value {:#?}", ret_val);
println!("");
```

From the code framework above, we can see that this challenge checks whether the player has successfully solved it by calling the `is_owner` function to check the `status` and evaluating the return value. 

```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/hellowWorld/framework/chall/sources/hello_world.move#L26-L38 */
// [*] Public functions
public entry fun answer_to_life(status: &mut Status, answer : vector<u8>) {
    // What is the answer to life?
    let actual = x"2f0039e93a27221fcf657fb877a1d4f60307106113e885096cb44a461cd0afbf";
    let answer_hash: vector<u8> = hash::blake2b256(&answer);
    assert!(actual == answer_hash, ERR_INVALID_CODE);
    status.solved = true;

}

public entry fun is_owner(status: &mut Status) {
    assert!(status.solved == true, 0);
}
```

From the contract source code, it's evident that setting the `solved` field in `status` to `true` requires calling the `answer_to_life` function. This function demands the user to provide the `status` and a `u8` vector called answer. It then hashes the answer using `blake2b256` and compares it to a predefined hash. If they match, the `solved` field is set to true. 


### Solution

This hash is clearly irreversible, but based on the hint in the comments, "What is the answer to life?" we can deduce that the answer is 42, as found in *The Hitchhiker's Guide to the Galaxy*.

```rust
module solution::hello_world_solution {
    use challenge::hello_world;
    public entry fun solve(status: &mut hello_world::Status) {
        let answer : vector<u8> = vector[52,50]; // ascii of "42"
        challenge::hello_world::answer_to_life(status, answer);
    }
}
```

## Challenge 2: Friendly Fire

{{< admonition note "Challenge Info" >}}
Keep your friends close and Enemies Closer.

- Host: metatrustctf.sui.io
- Port: 31338
- Source: https://github.com/MetaTrustLabs/ctf/tree/master/friendlyFire
- Score: 50
{{< /admonition >}}

### Target contract

```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/friendlyFire/framework/chall/sources/friendly_fire.move#L27-L40 */
// [*] Public functions
public(friend) fun get_flag(status: &mut Status) {
    status.solved = true;
}

public entry fun is_owner(status: &mut Status) {
    assert!(status.solved == true, 0);
}

public entry fun prestige(status: &mut Status, ctxSender: String, _ctx: &mut TxContext) {
    // let digest: &vector<u8> = tx_context::digest(_ctx);
    assert!(ctxSender == std::string::utf8(b"0x31337690420"), ERR_INVALID_CODE) ;
    get_flag(status);
}
```

This challenge is similar to the previous one, as it also requires us to set the `solved` field of `status` to true. However, in this challenge, the function that can modify the `status` is `get_flag`. Due to the restrictions imposed by the `friend` mechanism, we can only invoke `get_flag` through the `prestige` function. To do so, users need to provide `ctxSender` with the value `0x31337690420`.

### Solution

Since the contract doesn't perform any checks on the user input for `ctxSender` (which seems rather odd), all that is required is to provide the requested value.

```rust
module solution::friendly_fire_solution {
    use sui::tx_context::TxContext;
    use challenge::friendly_fire;

    public entry fun solve(status: &mut friendly_fire::Status, ctx: &mut TxContext) {
        challenge::friendly_fire::prestige(status, std::string::utf8(b"0x31337690420"), ctx);
    }
}
```

## Challenge 3: McChicken

{{< admonition note "Challenge Info" >}}
A customer just ordered from the secret menu, but none of the employees know how to cook the secret burgers. Could you please help?

- Host: metatrustctf.sui.io
- Port: 31339
- Source: https://github.com/MetaTrustLabs/ctf/tree/master/McChicken
- Score: 200
{{< /admonition >}}

### Target contract

This challenge presents an intriguing puzzle where the contract implements functions like `place_order` and `deliver_order` to simulate restaurant ordering and serving. A hamburger here can consist of five ingredients: Mayo, Lettuce, Chicken Schnitzel, Cheese, and Bun. 

```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/McChicken/framework/chall/sources/mc_chicken.move#L19-L23 */
struct Mayo has store, copy, drop { calories : u16 }
struct Lettuce has store, copy, drop { calories : u16 }
struct ChickenSchnitzel has store, copy, drop { calories : u16 }
struct Cheese has store, copy, drop { calories : u16 }
struct Bun has store, copy, drop { calories : u16 }

/* https://github.com/MetaTrustLabs/ctf/blob/master/McChicken/framework/chall/sources/mc_chicken.move#L69-L87 */
public fun get_mayo ( _chef: &mut ChefCapability ) : Mayo {
    Mayo { calories: 679 }
}

public fun get_lettuce ( _chef: &mut ChefCapability ) : Lettuce {
    Lettuce { calories: 14 }
}

public fun get_chicken_schnitzel ( _chef: &mut ChefCapability ) : ChickenSchnitzel {
    ChickenSchnitzel { calories: 297 }
}

public fun get_cheese ( _chef: &mut ChefCapability ) : Cheese {
    Cheese { calories: 420 }
}

public fun get_bun ( _chef: &mut ChefCapability ) : Bun {
    Bun { calories: 120 }
}
```

What makes it interesting is that when a user places an order for a hamburger, they provide a serialized byte sequence representing the hamburger. As chefs responsible for serving the orders, we need to deserialize the customer's order to obtain the recipe for the hamburger they desire.

### Solution

In the context of BCS encoding, there's an interesting piece of trivia to note: the serialization result for a struct is essentially the serialization result of all its fields combined. In our scenario here, when it comes to serializing a burger, it means concatenating several u16 calorie values that make up its various components together.

```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/McChicken/framework/src/main.rs#L84-L112 */
// Place Order1
let mut order_args : Vec<SuiValue> = Vec::new();
let order_args_1 = SuiValue::Object(FakeID::Enumerated(3, 0), None);
let recepit1 = Vec::from(
   [MoveValue::U8(0x78),
    MoveValue::U8(0x00),
    MoveValue::U8(0xa7),
    MoveValue::U8(0x02),
    MoveValue::U8(0x0e),
    MoveValue::U8(0x00),
    MoveValue::U8(0x29),
    MoveValue::U8(0x01),
    MoveValue::U8(0xa4),
    MoveValue::U8(0x01),
    MoveValue::U8(0x78),
    MoveValue::U8(0x00)]);
    order_args.push(order_args_1);
    order_args.push(SuiValue::MoveValue(MoveValue::Vector(recepit1)));

let ret_val = sui_ctf_framework::call_function(
    &mut adapter,
    chall_addr,
    "mc_chicken",
    "place_order",
    order_args,
    Some("customer".to_string())
).await;
println!("[SERVER] Return value {:#?}", ret_val);
println!("");
```

Taking Order1 as an example, its serialization result has a length of 12. Therefore, we can attempt to deserialize it into six u16 numbers:

```python
In [1]: import struct

In [2]: struct.unpack("<6H", b"\x78\x00\xa7\x02\x0e\x00\x29\x01\xa4\x01\x78\x00")
Out[2]: (120, 679, 14, 297, 420, 120)
```

Based on the calorie settings for each ingredient in the contract source code, we can deduce the recipe for the first burger as follows: Bun, Mayo, Lettuce, Chicken Schnitzel, Cheese, Bun. Similarly, we can derive the recipe for the second one. Using the `ChefCapability` permission to "create" the individual components of the hamburgers, we can represent both hamburgers using two wrapper structs and then proceed with the delivery of the orders.


```rust
module solution::mc_chicken_solution {
    // [*] Import dependencies
    use sui::tx_context::TxContext;
    use challenge::mc_chicken;

    struct Order1Burger has store, drop {
        bun: mc_chicken::Bun,
        mayo: mc_chicken::Mayo,
        lettuce: mc_chicken::Lettuce,
        chicken_schnitzel: mc_chicken::ChickenSchnitzel,
        cheese: mc_chicken::Cheese,
        bun2: mc_chicken::Bun,
    }

    struct Order2Burger has store, drop {
        bun: mc_chicken::Bun,
        cheese: mc_chicken::Cheese,
        cheese2: mc_chicken::Cheese,
        chicken_schnitzel: mc_chicken::ChickenSchnitzel,
        cheese3: mc_chicken::Cheese,
        chicken_schnitzel2: mc_chicken::ChickenSchnitzel,
        cheese4: mc_chicken::Cheese,
        chicken_schnitzel3: mc_chicken::ChickenSchnitzel,
        cheese5: mc_chicken::Cheese,
        cheese6: mc_chicken::Cheese,
        bun2: mc_chicken::Bun,
    }

    // [*] Public functions
    public fun solve(chef: &mut mc_chicken::ChefCapability, order1: &mut mc_chicken::Order, order2: &mut mc_chicken::Order, ctx: &mut TxContext) {
        let burger1 = Order1Burger {
            bun: mc_chicken::get_bun(chef),
            mayo: mc_chicken::get_mayo(chef),
            lettuce: mc_chicken::get_lettuce(chef),
            chicken_schnitzel: mc_chicken::get_chicken_schnitzel(chef),
            cheese: mc_chicken::get_cheese(chef),
            bun2: mc_chicken::get_bun(chef),
        };
        mc_chicken::deliver_order(chef, order1, burger1, ctx);

        let burger2 = Order2Burger {
            bun: mc_chicken::get_bun(chef),
            cheese: mc_chicken::get_cheese(chef),
            cheese2: mc_chicken::get_cheese(chef),
            chicken_schnitzel: mc_chicken::get_chicken_schnitzel(chef),
            cheese3: mc_chicken::get_cheese(chef),
            chicken_schnitzel2: mc_chicken::get_chicken_schnitzel(chef),
            cheese4: mc_chicken::get_cheese(chef),
            chicken_schnitzel3: mc_chicken::get_chicken_schnitzel(chef),
            cheese5: mc_chicken::get_cheese(chef),
            cheese6: mc_chicken::get_cheese(chef),
            bun2: mc_chicken::get_bun(chef),
        };
        mc_chicken::deliver_order(chef, order2, burger2, ctx);
    }
}
```

## Challenge 4: Coin Flip

{{< admonition note "Challenge Info" >}}
It's all about luck ... they say …

- Host: metatrustctf.sui.io
- Port: 31340
- Source: https://github.com/MetaTrustLabs/ctf/tree/master/coinFlip
- Score: 200
{{< /admonition >}}

### Target contract

In this challenge, the author has created a coin flipping game that requires users to consecutively guess correctly 12 times in a row. It's worth noting that the randomness of the coin flips is not provided through VRF (Verifiable Random Function) but is generated using a custom-defined LCG (Linear Congruential Generator) to produce random numbers.

```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/coinFlip/framework/chall/sources/coin_flip.move#L38-L50 */
public entry fun create_game( stake: Coin<SUI>, randomness: u64, fee: u8, ctx: &mut TxContext ) {
    let game = Game {
        id: object::new(ctx),
        stake: stake,
        combo: 0,
        fee: fee,
        player: RANDOM_ADDRESS,
        author: tx_context::sender(ctx),
        randomness: new_generator(randomness),
        solved: false,
    };
    transfer::public_share_object(game);
}

/* https://github.com/MetaTrustLabs/ctf/blob/master/coinFlip/framework/chall/sources/coin_flip.move#L90-L97 */
fun new_generator(seed: u64): Random {
    Random { seed }
}

fun generate_rand(r: &mut Random): u64 {
    r.seed = ((((9223372036854775783u128 * ((r.seed as u128)) + 999983) >> 1) & 0x0000000000000000ffffffffffffffff) as u64);
    r.seed
}
```

### Solution

If we carefully examine the code for creating the game within the framework, we can observe that the seed for the LCG is actually just a u8. Therefore, we can potentially predict the outcome of the coin flips by brute-forcing this seed, with a 1/256 chance of guessing the correct seed.

```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/coinFlip/framework/src/main.rs#L74-L82 */
let mut create_args : Vec<SuiValue> = Vec::new();

let mut rng = rand::thread_rng();
let random_byte: u8 = rng.gen();
println!("Random Seed: {}", random_byte);

let create_args_1 = SuiValue::Object(FakeID::Enumerated(3, 0), None);
let create_args_2 = SuiValue::MoveValue(MoveValue::U64(random_byte as u64));
let create_args_3 = SuiValue::MoveValue(MoveValue::U8(10));
```

Is there a more elegant way to obtain the seed without resorting to brute-force cracking? The answer is yes. Although Move language restricts us from accessing fields defined in the Foo module within the Bar module under normal circumstances, there's a clever workaround, which involves using the BCS encoding.

```rust
/* https://github.com/MetaTrustLabs/ctf/blob/master/coinFlip/framework/chall/sources/coin_flip.move#L22-L35 */
struct Random has drop, store, copy {
    seed: u64
}

struct Game has key, store {
    id: UID,
    stake: Coin<SUI>,
    combo: u8,
    fee: u8,
    player: address,
    author: address,
    randomness: Random,
    solved : bool,
}
```

Recalling what we mentioned earlier, BCS encoding only serializes and concatenates the underlying types within a struct. In the case of the `Game`, after serializing it, if we read the ninth-to-last byte (skipping the 1 byte for `solved` and the 7 high-order bytes for `seed`), we will obtain the least significant byte of `game.randomness.seed`. Once we have this seed, we can use the same LCG as in the challenge to generate random numbers and predict the outcomes, achieving a 100% correct guessing rate.

```rust
module solution::coin_flip_solution {

    // [*] Import dependencies
    use sui::tx_context::TxContext;
    use challenge::coin_flip;
    use sui::coin::{Self, Coin};
    use sui::sui::SUI;
    use std::bcs;
    use std::vector;
    struct Random has drop, store, copy {
        seed: u64
    }

    // [*] Public functions
    public entry fun solve(game: &mut coin_flip::Game, balance: Coin<SUI>, ctx: &mut TxContext) {
        let bytes: vector<u8> = bcs::to_bytes(game);
        let secret = *vector::borrow(&bytes, vector::length(&bytes) - 9);
        let r = new_generator((secret as u64));
        let round = 0;
        let fee = coin::split(&mut balance, 10, ctx);
        coin_flip::start_game(game, fee, ctx);
        while (round < 11) {
            let guess = generate_rand(&mut r) % 2;
            round = round + 1;
            coin_flip::play_game(game, (guess as u8), coin::split(&mut balance, 10, ctx), ctx);
        };
        let guess = generate_rand(&mut r) % 2;
        coin_flip::play_game(game, (guess as u8), balance, ctx);
    }

    fun new_generator(seed: u64): Random {
        Random { seed }
    }

    fun generate_rand(r: &mut Random): u64 {
        r.seed = ((((9223372036854775783u128 * ((r.seed as u128)) + 999983) >> 1) & 0x0000000000000000ffffffffffffffff) as u64);
        r.seed
    }

}
```
