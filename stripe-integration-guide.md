# Stripe Integration Guide

This document outlines basic details necessary for integrating Stripe into your application. It's important to consider these aspects to ensure smooth transaction processing.

## Currencies

When dealing with Stripe, it's crucial to remember that all amounts should be specified in a currency's smallest unit. For instance, in USD (United States Dollar), amounts are denoted in cents. Hence, to charge $10 USD, you should send the amount as 1000 (10 * 100).

For more information on currency handling and zero-decimal currencies, refer to the Stripe documentation on currencies: [Stripe Currency Documentation](https://docs.stripe.com/currencies#zero-decimal).

## Stripe Payments Test

During the testing phase, it's important to simulate transactions as closely as possible to real-world scenarios. To ensure that payments are realized and reflected in your Stripe account balance, you may need to bypass the pending balance. This is particularly relevant when using certain test cards, as funds from successful payments are otherwise directed to your pending balance.

Below is an example of how to bypass the pending balance using a specific test card:

| Description               | Number            | Details                                                                 |
|---------------------------|-------------------|-------------------------------------------------------------------------|
| Bypass pending balance    | 4000000000000077  | The US charge succeeds. Funds are added directly to your available balance, bypassing your pending balance. |
>https://docs.stripe.com/testing#available-balance

Utilizing the above test card in your integration testing will help mimic the direct addition of funds to your available balance, providing a more accurate testing environment.

For further details on testing with Stripe, including using other test cards for different scenarios, please consult the Stripe testing documentation: [Stripe Testing Documentation](https://docs.stripe.com/testing).
