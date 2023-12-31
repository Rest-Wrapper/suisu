---
# ZK Frontmatter
title: EPF Update -- Weeks 9, and 10
date: 2023-09-25 13:00:00
keywords: [epf, dev_update, blog]
---

[comment]: # (2023-09-04 to 2023-09-17)

# EPF Update -- Weeks 9, and 10 (Jordan Ellis Coppard)

Once again my predictions have fallen short. API Blueprint has next to no maintained tooling available for it and Bun with Elysia does not work for the project's needs. I discovered TypeBox which can output JSON Schema while providing runtime validation against that same schema-set which will be useful for the documentation generation.


## Hidden Problems

I thought it would be interesting to provide an example of how deceptively simple this kind of project can appear.

With the general roadblock from the last update overcome I took to implementing the _simplest_ REST endpoint the specification will have `/info/net` which returns:

```json
{
	peers: 42,
	chain: 1,
	listening: true,
}
```

This is an amalgamation of RPC `net_peerCount`, `net_version`, `net_listening`. Savvy readers may be screaming "no, not net_version!" and I indeed discovered [EIP-695](https://eips.ethereum.org/EIPS/eip-695) which is something I wasn't aware of before; this is one example of how something seemingly simple can have unexpected complications but not _the_ example I am building up to. While still using `net_version` I noticed the RPC library I am using (as there's no value in me implementing one for this project) [Viem](https://github.com/wagmi-dev/viem) incorrectly specifies the return type for `net_version` as an [EIP-1474 `QUANTITY`](https://eips.ethereum.org/EIPS/eip-1474#quantity) instead of a string as the specification requires. So, continually double-checking is _required_ lest errors creep in.

I'm using Elysia which itself uses TypeBox, and TypeBox can be used to construct the OpenAPI JSON Schema specification for the REST API. After dealing with a misrepresented type response for an RPC call using Viem I've elected to spend more time fleshing out using TypeBox as this demonstrates rigorous checking at all levels is required and so this takes up time even for "simple" things.

The real unexpected rabbit warren here though is uncovered when I asked myself two questions:

1. What _type_ can the values of chain id be?
2. What _range_ of chain id encompasses valid values?

Chain ids (as far as I can see) are all integers above zero. Is this constrained in a specification? Could I have a chain id of literally `foobarbaz`? If chain ids _are_ only allowed to be integers (by specification and not convention) what is the maximum permitted value? Surely a ludicrously high value... say a googol is not allowed?

Apparently (to me) there is no consensus for this yet. [EIP-2294](https://eips.ethereum.org/EIPS/eip-2294) proposes explicit bounds, [EIP-1344](https://eips.ethereum.org/EIPS/eip-1344) specifies an EVM opcode to return the chain id where the value is constrained to 256-bits. MetaMask constrains chain id to a much lower value [as at 2021-01-15](https://gist.github.com/rekmarks/a47bd5f2525936c4b8eee31a16345553); I am curious why MetaMask did-not/do-not use web standard [BigInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt) which can represent arbitrarily large integers. Finally there is on-going discussion on Ethereum Magicians relating to [EIP-2294](https://ethereum-magicians.org/t/eip-2294-explicit-bound-to-chain-id/11090) and [EIP-1344](https://ethereum-magicians.org/t/eip-1344-add-chain-id-opcode/1131).

I have decided to constrain the value to 256-bits and return it (in the REST API) as a BigInt to reflect the adopted EIP-1344 EVM opcode. I should check how widespread that adoption is (I only checked for Geth currently) but that's another problem that doesn't have an easy solution as far as I know. I'd like to be able to simply query a node to receive a list of EIPs it supports (as a standard). For now I peruse the source to determine whether an EIP is implemented or not.

So, wrapping three simple RPC methods turns into a debugging session (albeit a simple one) due to an incorrect type annotation, and a relatively deep dive into multiple EIP specifications to determine the bounded range the return value should have; and that's for the three simplest RPC methods to wrap.


## REST API

Using Elysia so far has been good, it has a nice API to require a specific request and response schema and can generate Swagger documentation for you. I've been reading up a bit more on TypeBox since that is the meat of how these schemas are actually defined and checked.
