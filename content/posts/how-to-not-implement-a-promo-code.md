---
author: ["Simeone Vilardo"]
title: "How to NOT implement a promo code"
date: "2024-03-17"
description: "How a promo code system can be easily reverse-engineered and exploited"
summary: "A promo code is a code used to participate in a particular promotion. Let’s consider a practical example. Imagine being a retail chain selling products, and we want to allow our customers to participate in a survey about the quality of our service. In exchange, they can receive a reward (such as a discount or a free product). However, we want this survey to be completed only by those who have actually made a purchase. To achieve this, we can print a unique code on the receipt, which the customer can use to participate in the survey. This code is the promo code."
tags: ["hacking", "programming"]
series: ["Reverse Engineering"]
---

## Introduction
What is meant by "promo code" in this article?
A promo code is a code used to participate in a particular promotion.
Let's consider a practical example. Imagine being a retail chain selling products, and we want to allow our customers to participate in a survey about the quality of our service. In exchange, they can receive a reward (such as a discount or a free product). However, we want this survey to be completed only by those who have actually made a purchase. To achieve this, we can print a unique code on the receipt, which the customer can use to participate in the survey. This code is the promo code.

## Look out for the front end
I personally encountered a case very similar to this several years ago (so I cannot guarantee it is still the case), and I won't disclose the company's name. The company allowed survey completion through a website.
The promo code was a 13-character alphanumeric string, for example: **`OCFBMEGDFXEEE`**
However, there was a major issue: the promo code verification logic was implemented within the front-end. Anyone inspecting the page source could easily trace a function that decrypted the promo code into some essential data and verified its validity through a checksum.
This leak is severe because it allows for the easy reversal of the decryption function and the generation of valid promo codes at will. It's simply a matter of "selecting" the values encoded within a promo code and calculating the checksum.
I understand the appeal of having a promo code that is "self-contained" and can thus return receipt information without needing any external source (like a database), but doing so requires a lot of attention and expertise. Moreover, it's always better to perform these operations on the back-end, even in cases where a database isn't required.

## Implementation
### Decrypt Function
Below is a pseudocode version of the decryption algorithm.
```js
function decryptPromoCode(code){
	var codeWithoutChecksum = removeChecksumFromCode(code);
	var decimalCode = convertBase28ToDecimal(codeWithoutChecksum);
	decimalCode = decimalCode.substring(1);
	decimalCode = reverseString(decimalCode);
	var siteId = Number(decimalCode.substring(0, 4));
	var posId = Number(decimalCode.substring(4, 6));
	var days = Number(decimalCode.substring(6, 10))
	var transId = Number(decimalCode.substring(10));
	return {days: days, siteId: siteId, posId: posId, transId: transId};
}
```
The function is straightforward and demonstrates that the promo code is simply a concatenation of:
- `siteId` (Store identification number)
- `posId` (Terminal identification number conducting the sale)
- `transId` (Transaction identification number)
- `days` (Number of days elapsed since December 31, 2024)
Each component of the promo code, `siteId`, `posId`, `transId`, and `days`, is padded with zeros on the left. For example, if the `siteId` were the number `42`, it would be expressed in the decimal promo code as `0042`.

The conversion from base 28 to base 10 is performed by the `convertBase28ToDecimal` function. It's a standard base conversion function, where the characters forming the base 28 are: `ABCDEFGHIJKLMNOPRSTUWXZ45679`.

The checksum is not relevant for our analysis. For completeness, I'll mention that it consists of 3 characters inserted within the promo code, respectively at positions `1`, `7`, and `11`. During decryption, the checksum is simply ignored. This makes sense because it can be verified at another time (before or after payload decoding).

Perhaps the most interesting aspect of this function is the line:
```js
decimalCode = decimalCode.substring(1);
```
Why is the first character removed during decryption? Is it unnecessary? Where does it come from?
We work with successive approximations and use empirical data to proceed. By decrypting various promo codes found on receipts, I can verify that this discarded character is always `1`. Therefore, we should keep in mind this peculiarity and try to write an encoding function.

### Encoding Function
By reversing the steps of the previous code, we obtain something like this:
```js
function generatePromoCode(days, siteId, posId, transId){
	var decimalCode = siteId + posId + days + transId + "1";
	var codeNoChecksum = convertToBase28(reverse(decimalCode));
	var checksum = generateChecksum(codeNoChecksum);
	var code = `${codeNoChecksum.charAt(0)}${checksum[2]}${codeNoChecksum.substring(1, 6)}${checksum[1]}${codeNoChecksum.substring(6, 9)}${checksum[0]}${codeNoChecksum.charAt(9)}`;
	return code;
}
```
As discussed earlier, we notice that we concatenate `1` with the other elements. The rest is straightforward and not particularly interesting.

## Severity Assessment
Now that we have both functions, we can put them to the test.
If I try to decode the example I provided earlier, **`OCFBMEGDFXEEE`**, I get the following result:
```js
{
	days: 1476,
	siteId: 37,
	posId: 6,
	transId: 5
}
```
If I encode this information using the inverse function, I indeed get **`OCFBMEGDFXEEE`**.
How severe is this? Quite severe. Anyone potentially can predict and generate any promo code. Sure, one might argue that you need to know the `siteId`, `posId`, and `transId`. However, as easily anticipated and experimentally verified, `posId` and `transId` are simply incremental numbers starting from 1. Each point of sale has a certain number of cash registers equivalent to `posId`, starting from the number `1`.
Similarly, `transId`, the transaction number, is a unique incremental for a certain POS on a certain date. The first receipt of POS number N will be `1`, and so forth.
The only slightly "complex" data to find might be the store ID. Certainly, we're talking about a 4-digit number and a chain with stores everywhere in my country, so almost any number could be valid. But what if we wanted to create a promo code for a specific store rather than just any store? Suppose the company's app (or used to have) a "find stores" section where there was a complete list of every store with its address. And, believe it or not, looking at the JSON of that list reveals that even if it's not shown to the user, the store ID related to each address is present there.

## It Can Get Even Worse? Yes
We've reached the juiciest part of our analysis. Shortly, I'll explain the mystery of the `1` that gets removed during decryption. But first, let's imagine a use case for this promo code system (I assure you this use case went into production for a long period where I live).

> Every user who purchases item A can receive 1 point by entering the promo code related to that purchase on our web app. You can make a maximum of 5 entries per day. When you accumulate 10 points, you can redeem item B for free.
> NOTE: If an entry is made with a promo code whose receipt does not include the purchase of item A, that entry is still counted toward the daily 5-entry limit.

A malicious attacker could generate valid promo codes, hoping that some of them are linked to a receipt that actually includes the purchase of item A. However, there is no way to know, before entering it into the system, if this is true. Furthermore, the limit of 5 entries per day makes it extremely disadvantageous to attempt brute force. In most cases, even creating multiple accounts, each account would be immediately blocked for that day, consuming all 5 entries in vain.

If only it were possible to create "collisions," that is, generate two or more promo codes whose alphanumeric characters are different but whose encoded information inside is the same, then brute force could be viable! It would be sufficient to try a large number of promo codes using a large number of accounts. The promo codes containing item A would give 1 point to that account, which would still be wasted (because it would not be possible to load more points for the entire day). However, you would obtain a list of promo codes (burned because they have already been entered into the system) that actually include item A. By creating collisions of such codes, you could enter these collisions into a new account, which could redeem 5 points in a single day and get item B for free in two days in 100% of cases (there would never be cases of promo codes with zero points). But how can collisions be generated? By now, you've probably guessed it's thanks to our `1`.

## Collisions: How and Why
Let's see what happens during the generation of a promo code with the following information to encode:
```js
{
	days: 666,
	siteId: 90,
	posId: 69,
	transId: 420
}
```
If we concatenate this information (with appropriate padding of zeros), we would get the string `00906906660420`.
Reversing this string, we get `02406660960900`.
This number, when converted to base 28, becomes `GKKF49BLA`.
Here's the interesting part: when converted to base 10, this number becomes `2406660960900`.
We've just lost a `0`. Losing a `0` obviously ruins the entire decryption algorithm, which relies on the assumption that each variable has a fixed length of characters.
This is why the `1` is added during encoding! It's done to handle the case where the transaction ID ends with a `0`, a value that would be lost during conversion to an integer because leading zeros are not significant digits.
But this means... we've just discovered a way to provoke collisions for every promo code in which the transaction ID is not a multiple of 10.
We can indeed modify the encoding function as follows:
```js
function generatePromoCode(days, siteId, posId, transId, collisionChar){
	var decimalCode = siteId + podId + days + transId + collisionChar;
	var codeNoChecksum = convertToBase28(reverse(decimalCode));
	var checksum = generateChecksum(codeNoChecksum);
	var code = `${codeNoChecksum.charAt(0)}${checksum[2]}${codeNoChecksum.substring(1, 6)}${checksum[1]}${codeNoChecksum.substring(6, 9)}${checksum[0]}${codeNoChecksum.charAt(9)}`;
	return code;
}
```
And invoke this function twice:
```js
var promoCode1 = generatePromoCode(1476, 37, 6, 5, "0");
var promoCode2 = generatePromoCode(1476, 37, 6, 5, "1");
console.log(promoCode1); // "EHWKEBGSDW9HR"
console.log(promoCode2); // "OCFBMEGDFXEEE"
```
The two generated promo codes, namely **`EHWKEBGSDW9HR`** and **`OCFBMEGDFXEEE`**, are both valid, correctly decoded by the official decoding function, and contain the same information!
It is also possible to exploit this approach to try a second collision, in this case valid even for transactions with IDs ending in `0`. You must have already guessed it; just use `2` as the final character. In this case, we have:
```js
var promoCode3 = generatePromoCode(1476, 37, 6, 5, "2");
console.log(promoCode3); // "4MSWWHGTHXIPW"
```
Our new promo code **`4MSWWHGTHXIPW`** also encodes the same information. It is possible to continue and generate further collisions at least until using the char `9`? Unfortunately (or fortunately), it is not possible. Any attempt to generate promo codes with `collisionChar` greater than 2 will corrupt the encoded data. Why? Simple, because the `collisionChar` will inevitably end up as the first place on the left of the decimal number. Since this decimal number will be converted to base 28 and result in a string length of 10 characters, it must necessarily be less than or equal to `296196766695423` (28^10).
Having `3` as the first numerical value will result in a number that, when converted to base 28, is 11 characters long.

## What's the moral?
When creating these systems, make sure that the generated codes:
- Are not predictable and reproducible
- Cannot generate collisions
In my opinion, it would be nice to be able to generate self-contained codes, but I'm not sure if it's possible. JWT tokens are a great example; thanks to their digital signature mechanism, you can trust their content and origin. However, if such a token needs to be manually used by the user, it should be as short and easy as possible, so embedding the signature within the token doesn't seem feasible.
Let's not skimp. Let's use tables and let's use API calls. A simple random alphanumeric string of 8 characters allows for `2821109907456` combinations. That's almost three trillion. And being random, good luck reproducing a code or generating a collision.