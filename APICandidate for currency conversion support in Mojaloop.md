Currency conversion support in Mojaloop

# Introduction

This document describes a proposal for supporting currency conversion in a Mojaloop switch. It includes cases where either the payee DFSP or the payer DFSP is capable of operating in several currencies, and hence of performing currency conversion for transfers internally.

## Definitions of terms

The following table provides definitions of terms used in this document.

| Term | Definition |
|------|------------|
| FXP  |            |

## References

| Reference | Description                                      | Version |
|-----------|--------------------------------------------------|---------|
|           | Open API for FSP Interoperability Specification  | 1.1     |
|           | Draft PISP Specification changes                 | 1.5     |

## Version history

| Version | Description                                     | Modified By | Date             |
|---------|-------------------------------------------------|-------------|------------------|
| 1.0     | Initial version                                 | M. Richards | 4 June 2021      |
| 1.1     | Following FSPIOP SIG review                     | M. Richards | 25 June 2021     |
| 1.2     | Candidate for adoption                          | M. Richards | 6 August 2021    |
| 1.3     | Following Review                                | M. Richards | 23 August 2021   |
| 2.0     | Architectural simplification                    | M. Richards | 2 September 2021 |
| 2.1     | Following reviews                               | M. Richards | 29 November 2021 |
| 2.2     | Following FSPIOP SIG presentation               | M. Richards | 3 December 2021  |
| 2.3     | Picking up the threads                          | M. Richards | 22 May 2023      |
| 2.4     | Remove financial information from message       | M. Richards | 21 June 2023     |
| 2.5     | Manage fulfilment as part of transaction object | M. Richards | 22 June 2023     |
| 2.6     | Post convening presentation and review          | M. Richards | 10 July 2023     |
| 2.7     | Add reference currency support                  | M. Richards | 4 August 2023    |
|         |                                                 |             |                  |

# Assumptions

1.  We assume the existence of an FX API: a separate API which participants in a Mojaloop scheme can use to access the facilities required for currency conversion.
2.  We assume that the list of currencies in which a payee’s account(s) can receive is returned as part of the discovery process (i.e. by an existing **PUT /parties** call in the FSPIOP API.)
3.  We assume that the description of the transaction’s terms which is given in the body of the **POST /quotes** call and included in the **Transaction** object in the **PUT /quotes** and **POST /transfers** calls will allow the definition of a send amount and a receive amount separately.
4.  We assume that a version of the **/services** resource defined in the PISP API will allow a DFSP to identify participants in the scheme which offer FXP services, but that this service will be provided as part of the FX API.

# Use cases

We need to support at least the following use cases:

## Self support

It is possible that a participant may be able to provide its own currency conversion service, even though the account which it is representing is not able to send or receive in the currency specified. Liquidity cover for the transfer is provided in the target currency by the Payer DFSP, and the Payer DFSP settles with the Payee DFSP in the target currency. The scheme will require the participant to maintain liquidity support in the target currency using the existing liquidity support functionality.

We assume that this use case can be supported using the existing FSPIOP API.

## Currency purchase

Where a participant needs to obtain an amount of a particular currency to support transfers, it can do so by purchasing an amount of the foreign currency from a specialised provider (an *FXP*) at an agreed rate. “Purchasing” here means: liquidity cover for the transaction in the source currency is provided by the FXP, and the FXP settles with the Payee DFSP in the target currency. The FXP may also provide liquidity cover for the converted funds and, if it does, it should receive liquidity credit for this.

## PvP

A participant may enlist the FXP as a participant in the transaction. In this case, the transaction consists of two : one between the payer DFSP and the FXP in the source currency, and one between the FXP and the payee DFSP in the target currency. There are two settlements, one for each currency movement.

The first of these use cases does not require the involvement of an FXP; the others do.

# Questions of design

## Who should identify the requirement for currency conversion?

Existing implementations of Mojaloop FX have relied on the Mojaloop switch to analyse the content of a proposed transfer and decide whether the transfer requires currency conversion or not. This pattern has the advantage of simplifying life for a DFSP.

The alternative is to allow one of the DFSP participants to decide whether a proposed transfer requires currency conversion or not. The information required to make this decision will be available to the Payer DFSP following the discovery phase, since the Payee DFSP will return the currencies that the account owner can receive transfer in as part of the **PUT /parties** response (by assumption 2 above); and it will also be included in the quotation request issued by the payer DFSP, since otherwise the switch could not identify it. Either the payer DFSP or the payee DFSP could therefore identify the requirement for currency conversion.

Overall, it seems simpler to allow a DFSP to decide whether to engage an FXP or not. This pattern is also closer to our general inclination to leave things to the participants where possible.

## Who should request currency conversion?

Existing implementations and designs have relied on a model in which either the switch (in Mowali) or the debtor DFSP sends a **POST /quotes** request to the FXP. The FXP then generates a new **POST /quotes** request in the target currency (in Mowali), or adds a leg to the existing transaction, and forwards the request to the Creditor DFSP.

An alternative approach (and the one described in this proposal) is to allow the principals in the transfer to continue as the sole parties to the transfer, as they do in transfers where no currency conversion is required, and to allow them to request currency conversion from an FXP as required.

## When and how should the terms of currency conversion be finalised?

Existing implementations and designs have included the commitments made by an FXP as part of the request part of the agreement process. This has either been by breaking the transfer into two separate transfers (Mowali) or by adding information to the initial quotation request before it is passed to the payee DFSP.

This approach has the drawback that the FXP is being asked to make a commitment about the transfer before its terms are definitively set. The terms of the transfer are finally set by the payee DFSP, and it is therefore possible for the FXP to commit to a conversion which is subsequently changed by the payee DFSP, for instance by the addition of fees. If, however, the FXP wants to change the send or receive amounts to reflect the new agreement, then the payee DFSP’s condition and fulfilment will be invalidated.

The approach proposed in this document splits the currency conversion process entirely from the transfer process. The transfer proposed to Mojaloop is always in a single currency, and currency conversion may be carried out either by the payer DFSP or the payee DFSP. If currency conversion is being performed by the payer DFSP, then:

-   If the amount is a SEND amount, then the payer DFSP will need to obtain a currency conversion quotation from the FXP before sending the Mojaloop request for quotation, and will use the amount in the target currency returned from the currency conversion quotation as the **amount** field in the Mojaloop quotation request.
-   If the amount is a RECEIVE amount, then the payer DFSP will have been informed of the amount to be received before it makes the Mojaloop quotation request, and will use that amount in the Mojaloop quotation request.

If the payer DFSP decides to not to undertake the currency conversion, then it requests a quotation from the payee DFSP using the source currency. At this point, the payee DFSP will have two options. If it does not want to undertake the currency conversion, then it can reject the request for quotation with an appropriate error response. Otherwise, it can request currency conversion itself, using the amount in the source currency sent through in the request for quotation.

## How should the FXP ratify the transfer?

At present, the Mowali implementation creates two separate transactions if currency conversion is required. Whatever the drawbacks of this approach, it does mean that the FXP has the chance to participate in the ILP signature process. In other designs for this area, the FXP has had to rely on the payee DFSP’s signature of the terms of the transfer, and has not had the opportunity to give independent confirmation of the validity of its part of the transfer.

This design proposes an alternative approach. It proposes that, when the FXP confirms the terms under which it intends to facilitate those transfers (see Section 4.3 above,) it should sign those terms using its own private key, produce a condition and return the condition (and an expiry) as part of its response. We then propose a separate API call by which a participant (or the switch) can ask the FXP to execute a set of conversion terms and return the appropriate fulfilment.

A positive response from the FXP to the currency conversion execution request means: I have approved the transfer and assent to its final determination by the switch. The switch should not ratify a transfer which includes currency conversion unless it can satisfy itself that the FXP has assented to the transfer by returning the fulfilment which matches the condition that the switch holds.

# Support for a reference currency

In some cases, an FXP will provide a direct conversion between the currencies of the debtor and creditor accounts. In other cases, however, this will not be possible: for instance, if the transfer is being executed between two currencies which are widely separated geographically and/or relatively illiquid. In cases like this, it may be necessary to convert from the source currency to a reference currency (such as USD, for instance) using one FXP, and from the reference currency to the target currency using another FXP. This use case can be supported using the existing functionality of the currency conversion API.

There is a further use case to support. In this use case, a single FXP is used, but the transfer is routed through a reference currency in order to support architectures where a meta-scheme is used to allow all parties to settle between each other using a single reference currency. In this use case, currency pre-purchase should not be supported.

This use case can also use existing functionality, and will be able to support currency conversion initiated by the payee for a RECIVE amount. This will work in the following way:

1.  The debtor DFSP requests a quotation, giving the required amount in the target currency and specifying that it can send in its source currency.
2.  The creditor DFSP requests conversion from the source currency to the target currency.
3.  The FXP, which knows that it needs to use a reference currency, overrides the source currency requested by the creditor DFSP and returns a quotation for a conversion from the reference currency to the target currency.
4.  The creditor DFSP approves its quotation based on the conversion and returns it to the debtor DFSP. The amount of the transfer will now be expressed in the reference currency.
5.  The debtor DFSP registers that it too will need to obtain currency conversion so that it can send in the reference currency. It asks for a conversion from the source currency to the reference currency. Note that this conversion does not form part of the terms of the transfer, and there is therefore no need for the transfer to be re-validated by the creditor DFSP.
6.  When the terms of the currency conversion are approved, the debtor DFSP knows how much will be debited from its customer’s account. It can show all the information relating to the payment to its customer, and the customer can approve.
7.  When the customer approves the payment, the debtor DFSP executes the currency conversion, specifying that the conversion is dependent on the success of the associated transfer. The switch makes a reservation in the source currency against the account of the debtor DFSP.
8.  On successful execution of the currency conversion, the debtor DFSP requests execution of the transfer, which is denominated in the reference currency. The switch makes a reservation in the reference currency against the account of the FXP.
9.  On receipt of the payment execution request, the creditor DFSP requests execution of the conversion to the target currency, specifying that the conversion is dependent on the success of the associated transfer. The switch makes a reservation in the target currency against the account of the second FXP.
10. On successful execution of the currency conversion, the creditor DFSP approves the transfer.
11. The switch assigns obligations using the following algorithm:
    1.  Does the credit party to the transfer have an account in the currency of the transfer?
        1.  If so, create an obligation between the party of the reservation for the transfer and the credit party to the transfer.
        2.  If not, is there a dependent transfer whose credit party is the credit party for the transfer?
            1.  If so, create an obligation between the party of the reservation for the transfer and the debit party for this dependent transfer.
            2.  If not, this is an error.
    2.  For each dependent transfer associated with the main transfer:
        1.  Create an obligation between the party of the reservation for the transfer and the counterparty to the transfer.

# Functions of an FXP

An FXP needs to be able to support the following functions:

## Currency pre-purchase

A DFSP sends a request to an FXP to purchase a specified amount of a currency, using the **fxQuotes** endpoint described in Section 6.3.2 below. The DFSP specifies the amount of either the source currency or the target currency to be purchased, and the account type to be used for the position account associated with pre-purchased currency.

The FXP responds with an amount of the opposite currency, an expiry date for the purchase, and a condition which the DFSP can use to execute the purchase.

If the DFSP decides to go ahead with the purchase, it sends a request for execution of the purchase using the fxTransfers endpoint described in Section 6.3.3 below. The FXP waits until it can satisfy itself that the funds requested have been made available to it, and it responds with the fulfilment of the condition. This says to the DFSP that the funds have been accepted and cleared to its account in the switch. The funds will be available immediately for liquidity cover if this is the account used for general transfers. Otherwise, the DFSP will be able to transfer the funds to any other settled funds account once they have been settled.

## Transaction agreement

We allow either party to a transfer to request currency conversion, and optionally to associate the currency conversion agreement to a transfer, such that the currency conversion will only succeed if the associated transfer also succeeds.

### Agreement request

When the FXP receives a conversion request, it fills in the **target.amount.principalAmount** field (if this is blank) or the **source.amount.principalAmount** field (if this is blank.) If both fields are blank, or if both fields contain a value, then an error is returned. The requester can specify a time for which they want the currency conversion to be valid.

If the conversion can be approved, then the FXP signs the **conversion** object and creates a condition from the signature. It adds an expiry time (which need not be later than the expiry time proposed by the requester,) and returns the resulting **conversion** object to the caller.

## Transaction execution

The execution of a transfer for which currency is conversion is required will depend on whether the payer DFSP or the payee DFSP intends to undertake the currency conversion. The different scenarios are described in more detail below.

### Transaction execution when conversion is performed by the payer DFSP

When the payer DFSP is undertaking currency conversion, transfer execution proceeds in the following way:

1.  The payer DFSP requests execution of the currency conversion request which it has previously issued to the FXP.
    1.  Liquidity cover for the currency conversion request, if required, will be in the source currency and is provided by the payer DFSP in the normal way.
    2.  The payer DFSP will specify as part of the execution request that the conversion is dependent on the execution of a transfer by giving the transfer ID which it will use to identify the transfer. Confirmation of the conversion by the FXP implies that it will accept the determination of the transfer by the switch.
    3.  Since completion of the conversion is dependent on the outcome of the transfer, the switch’s currency conversion processor will monitor the output stream from the transfer fulfilment process in the switch, waiting for information about the final state of the transfer.
    4.  When the FXP confirms the conversion, the currency conversion processor in the switch will transfer the amount in the target currency from the FXP’s position account to the payer’s position account. This will enable the payer DFSP to meet the switch’s requirements for liquidity cover when it requests the transfer, and is equivalent to committing the conversion.
    5.  The amount of the transfer will be expressed in the target currency.
    6.  The transfer ID will be the same as that given to the FXP as part of the currency conversion execution request.
2.  The payer DFSP will issue a **POST /transfers** in the normal way, using the target currency. Liquidity cover for the transfer will be provided as a consequence of the currency conversion process, as described in item 1)d) above.
3.  The payee DFSP will receive the transfer request in the normal way. If a payer DFSP manages currency conversion, a payee DFSP will not need to know that the transfer involves currency conversion.
4.  When the payee DFSP approves the transfer and returns its response to the switch, the switch will behave as follows:
    1.  It will record a transfer between the payer and the payee DFSP in the target currency.
    2.  The switch’s transfer fulfilment handler will put a message on its output stream to confirm the success of the transfer.
    3.  The switch’s currency conversion processor will pick up the message from the output stream and use it to inform the FXP of the outcome of the transfer.
        1.  If the transfer has succeeded, then:
            1.  The switch will commit the funds reserved in step 1)a) above, and will balance them with a credit to the FXP’s position account in the source currency. This is the currency conversion equivalent to committing the funds transfer in a standard transfer without currency conversion.
            2.  The other part of the currency conversion, which involves a debit to the FXP and a credit to the payer DFSP in the target currency, was already made in step 1)d) above. No further action needs to be performed by the switch.
            3.  The result will be three matching pairs of ledger entries, all of which will share a transaction ID, as follows:
                1.  A debit from the payer DFSP and a credit to the FXP in the source currency;
                2.  A debit from the FXP and a credit to the payer DFSP in the target currency;
                3.  A debit from the payer DFSP to the payee FXP in the target currency.
            4.  The currency conversion processor will send a **PATCH /fxTransfers** message to the FXP to confirm that the switch has recorded the conversion and that the FXP can clear the funds in its ledgers.
        2.  If the transfer has failed, then:
            1.  The switch’s currency conversion processor will cancel the reservation created in step 1)a) above in its ledgers. The reservation created against the payer DFSP in the target currency will already have been cancelled by the transfer fulfil handler.
            2.  The switch’s currency conversion processor will cancel the transfer made in step 1)d) above. This will return the FXP’s funds to it.
            3.  The currency conversion processor will send a **PATCH /fxTransfers/\<ID\>/error** message to the FXP to confirm that the transfer has failed and that the FXP should cancel the conversion in its ledgers.
5.  The switch will inform the payer DFSP of the outcome of the transfer.
6.  TODO: how do we support immediate gross settlement when currency conversion is involved?

### Transaction execution when conversion is performed by the payee DFSP

When the payee DFSP is undertaking currency conversion, transfer execution proceeds in the following way:

1.  The payer DFSP sends a request for quotation using the existing structure. The amount will be specified in the source currency.
    1.  If the payer DFSP has specified that the amount type is RECEIVE, this is an error. The payee DFSP should reject the request. TODO: how would we add support for RECEIVE requests where the payer DFSP wants the payee DFSP to perform the currency conversion?
    2.  If the payee DFSP does not want to, or cannot, perform the currency conversion itself, then it should reject the request with an appropriate error code.
2.  The payee DFSP asks the FXP to quote for the currency conversion based on the information provided by the payer DFSP, plus any additional information that it would like to include (e.g. additional fees.)
3.  If the FXP confirms a response, and the payee DFSP decides to go ahead, then the payee DFSP should:
    1.  Ensure that its quote expiry date is not later than the conversion expiry date provided by the FXP.
    2.  Fill in the **payeeReceiveAmount** field in the quotation response with the appropriate value in the target currency.
    3.  Add any fees charged by the FXP to the fees charged in the transfer, if appropriate.
    4.  Respond positively to the quotation request.
4.  When the payer DFSP requests execution of the transfer, the amount of the transfer will be given in the source currency.
    1.  The switch’s transfer prepare handler will reserve funds for the transfer in the source currency.
5.  The payee DFSP requests execution of the currency conversion request which it has previously issued to the FXP.
    1.  The payee DFSP will specify as part of the execution request that the conversion is dependent on the execution of a transfer by giving the transfer ID which it will use to identify the transfer. Confirmation of the conversion by the FXP implies that it will accept the determination of the transfer by the switch.
    2.  If liquidity cover is required for the currency conversion, then the switch’s currency conversion processor will check to see whether the currency conversion is linked to a transfer ID.
        1.  If it is not, then the currency conversion processor will reserve funds on the requester’s account in the source currency.
        2.  If it is, the currency conversion processor will check to see whether funds have been reserved under this transfer ID (that is, whether the payer DFSP has already committed irrevocably to the transfer.)
            1.  If funds have already been reserved, then the currency conversion processor will make a reservation on the payee DFSP’s account in the source currency, but will not reject the conversion if the payee DFSP’s account does not have liquidity cover for the conversion.
            2.  If funds have not been reserved, then the currency conversion processor will make a reservation against the payee DFSP’s account in the source currency in the normal way.
    3.  Since completion of the conversion is dependent on the outcome of the transfer, the switch’s currency conversion processor will monitor the output stream from the transfer fulfilment process in the switch, waiting for information about the final state of the transfer.
6.  The payee DFSP confirms the transfer to the switch using a **PUT /transfers** call.
7.  The switch’s transfer fulfil handler performs the following actions:
    1.  It confirms a debit from the payer DFSP in the source currency, and a credit to the payee DFSP in the source currency.
    2.  It places the confirmation message on its output stream for transmission to the payer DFSP.
8.  The switch’s currency conversion processor will pick up the message and confirm that the transfer ID is one which it is listening for. It will perform the following actions:
    1.  It removes the reservation created in step 5)b) above.
    2.  It fulfils the currency conversion by:
        1.  Creating a debit entry in the payee DFSP’s account and a credit entry in the FXP’s account in the source currency.
        2.  Creating a debit entry in the FXP’s account in the target currency and a credit entry in the payee DFSP’s account in the target currency.
    3.  The result will be three matching pairs of ledger entries, all of which will share a transaction ID, as follows:
        1.  A debi from the payee DFSP and a credit to the FXP in the source currency;
        2.  A debit from the FXP and a credit to the payee DFSP in the target currency;
        3.  A debit from the payer DFSP to the payee FXP in the source currency.
    4.  It sends a **PATCH /fxTransfers** message to the FXP so that the FXP can clear the transfer in its ledgers.
9.  The switch will forward the confirmation of the transfer to the Payer DFSP.

# Proposed API changes

## FSPIOP API

The following changes will be made to the existing FSPIOP API:

### Changes to the **PUT /parties** response

The **PUT /parties** response (see Section 6.3.4.1 of **1.2 above**) will be modified to return information about the currencies in which a beneficiary can receive funds.

This will involve changes to the **party** object (see Section 7.4.11 of **1.2 above**) to include information about the currencies available. It would be possible to do this, for instance, by including an optional array of **account** objects (as defined in Section 3.2.1.1 of **Ref 2 above**) containing the currencies in which the beneficiary can receive.

The proposed new version of the **party** object, replacing table 90 of **1.2 above**, will be as shown below:

| **Name**                       | **Cardinality** | **Type**                                    | **Description**                                                                                                 |
|--------------------------------|-----------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| **partyIdInfo**                | 1               | [PartyIdInfo](#_bookmark362)                | Party Id type, id, sub ID or type, and FSP Id.                                                                  |
| **merchantClassificationCode** | 0..1            | [MerchantClassificationCode](#_bookmark307) | Used in the context of Payee Information, where the Payee happens to be a merchant accepting merchant payments. |
| **name**                       | 0..1            | [PartyName](#_bookmark317)                  | Display name of the Party, could be a real name or a nick name.                                                 |
| **personalInfo**               | 0..1            | [PartyPersonalInfo](#_bookmark364)          | Personal information used to verify identity of Party such as first, middle, last name and date of birth.       |
| **supportedCurrencies**        | 0..16           | Currency                                    | Up to 16 currencies in which the party can receive funds.                                                       |

### Changes to the **POST /quotes** message

It would be possible to use the existing data structures of the quotation request to allow the payee DFSP to infer whether it should request currency conversion or not. Analysis of the currency in which the quotation request is reliable except for one case: that in which the sender wants the recipient to receive a specified amount of the target currency, but the payer DFSP does not want to undertake the currency conversion. In this case, the amount of the transfer would be expressed in the target currency and the *amountType* would be set to RECEIVE.

To accommodate this case, we propose to add an optional field to the **POST /quotes** data model described in table 22 of **1.2 above**. This field will allow the payer DFSP to specify which DFSP it wants to undertake currency conversion. This field will be a text enumeration whose name will be *converter*. It will be a value from the enumeration **CurrencyConverter**, as described in Section 6.2.1 below.

The modified form of the **/quotes** request object will be as given below:

| **Name**                 | **Cardinality** | **Type**                                | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
|--------------------------|-----------------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **quoteId**              | 1               | [CorrelationId](#_bookmark281)          | Common ID between the FSPs for the quote object, decided by the Payer FSP. The ID should be reused for resends of the same quote for a transaction. A new ID should be generated for each new quote for a transaction.                                                                                                                                                                                                                                                                                                     |
| **transactionId**        | 1               | [CorrelationId](#_bookmark281)          | Common ID (decided by the Payer FSP) between the FSPs for the future transaction object. The actual transaction will be created as part of a successful transfer process. The ID should be reused for resends of the same quote for a transaction. A new ID should be generated for each new quote for a transaction.                                                                                                                                                                                                      |
| **transactionRequestId** | 0..1            | [CorrelationId](#_bookmark281)          | Identifies an optional previously-sent transaction request.                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **payee**                | 1               | [Party](#_bookmark358)                  | Information about the Payee in the proposed financial transaction.                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **payer**                | 1               | [Party](#_bookmark358)                  | Information about the Payer in the proposed financial transaction.                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **amountType**           | 1               | [AmountType](#_bookmark267)             | **SEND** for send amount, **RECEIVE** for receive amount.                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **amount**               | 1               | [Money](#_bookmark356)                  | Depending on **amountType**: If **SEND**: The amount the Payer would like to send; that is, the amount that should be withdrawn from the Payer account including any fees. The amount is updated by each participating entity in the transaction. If **RECEIVE**: The amount the Payee should receive; that is, the amount that should be sent to the receiver exclusive any fees. The amount is not updated by any of the participating entities.                                                                         |
| **converter**            | 0..1            | [CurrencyConverter](#currencyconverter) | **PAYER** if the payer DFSP intends to perform currency conversion; **PAYEE** if the payer DFSP wants the payee DFSP to perform currency conversion. If absent and the transfer requires currency conversion, then the converting institution will be inferred from the currency in which the quotation is requested: if the quotation is requested in the source currency, then the payee DFSP should perform currency conversion. If it is in the target currency, then the payer DFSP will perform currency conversion. |
| **conversionRate**       | 0..1            | [FxRate](#fxrate)                       | Used by the debtor party if it wants to share information about the currency conversion it proposes to make; or if it is required by scheme rules to share this information. This object contains the amount of the transfer in the source and target currencies, but does not identify the FXP being used.                                                                                                                                                                                                                |
| **fees**                 | 0..1            | [Money](#_bookmark356)                  | Fees in the transaction. The fees element should be empty if fees should be non-disclosed. The fees element should be non-empty if fees should be disclosed.                                                                                                                                                                                                                                                                                                                                                               |
| **transactionType**      | 1               | [TransactionType](#_bookmark372)        | Type of transaction for which the quote is requested.                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **geoCode**              | 0..1            | [GeoCode](#_bookmark354)                | Longitude and Latitude of the initiating Party. Can be used to detect fraud.                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **note**                 | 0..1            | [Note](#_bookmark311)                   | A memo that will be attached to the transaction.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **expiration**           | 0..1            | [DateTime](#_bookmark255)               | Expiration is optional. It can be set to get a quick failure in case the peer FSP takes too long to respond. Also, it may be beneficial for Consumer, Agent, and Merchant to know that their request has a time limit.                                                                                                                                                                                                                                                                                                     |
| **extensionList**        | 0..1            | [ExtensionList](#_bookmark344)          | Optional extension, specific to deployment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

### Changes to the **transaction** object

If the payer DFSP requests that the payee DFSP perform a currency conversion, the payee DFSP is not under an obligation to accept this request. The general principle of the FSPIOP API is that the payee DFSP should be in charge of the terms of a transfer. Since this is so, it will be necessary for the payee DFSP optionally to include a determination of the entity which will undertake the currency conversion in the **transaction** object described in table 96 of **1.2 above**.

The modified form of the **transaction** object will therefore be as given below:

| Name               | Cardinality | Type                                    | Comment                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
|--------------------|-------------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| transactionId      | 1           | CorrelationId                           | ID of the transaction, decided by the Payer FSP during the creation of the quote                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| quoteId            | 1           | CorrelationId                           | ID of the quote request, decided by the Payer FSP during the creation of the quote                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| payee              | 1           | Party                                   | Information about the Payee in the proposed financial transaction.                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| payer              | 1           | Party                                   | Information about the Payer in the proposed financial transaction                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| amount             | 1           | Money                                   | The transaction amount to be sent.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| payeeReceiveAmount | 0..1        | Money                                   | The amount that the beneficiary will receive.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| transactionType    | 1           | TransactionType                         | Type of the transaction.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| converter          | 0..1        | [CurrencyConverter](#currencyconverter) | **PAYER** if the payer DFSP intends to perform currency conversion; **PAYEE** if the payer DFSP wants the payee DFSP to perform currency conversion. If absent and the transfer requires currency conversion, then the converting institution will be inferred from the currency in which the quotation is requested: if the quotation is requested in the source currency, then the payee DFSP should perform currency conversion. If it is in the target currency, then the payer DFSP will perform currency conversion. |
| currencyConversion | 0..1        | [FxRate](#fxrate)                       | Contains information about the currency conversion proposed by the payer DFSP, if the payer DFSP wants to share it or if scheme rules require it to be shown.                                                                                                                                                                                                                                                                                                                                                              |
| note               | 0..1        | Note                                    | Memo associated with the transaction intended for the Payee                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| extensionList      | 0..1        | ExtensionList                           | Optional extension list, specific to the deployment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

## New data objects in the FSPIOP API

The following data object will be added to the FSPIOP API to allow participants to make requests to each other to support currency conversion.

### CurrencyConverter

The following table contains the allowed values for the enumeration **CurrencyConverter**.

| Name  | Description                                          |
|-------|------------------------------------------------------|
| PAYER | Currency conversion should be performed by the payer |
| PAYEE | Currency conversion should be performed by the payee |

### Dependent

The **Dependent** object contains information about a participant in the transfer other than the debtor FI and the creditor FI. In this case, the participants will be the FX provider(s) who are supporting the transfer. The **Dependent** object will have the structure described below.

| Name         | Cardinality | Type         | Comment                                                  |
|--------------|-------------|--------------|----------------------------------------------------------|
| intermediary | 1           | FspId        | The intermediary whose obligations are being registered. |
| condition    | 1           | IlpCondition | ILP condition for the intermediary operation.            |

### FxRate

The **FxRate** object contains information about a currency conversion in the transfer. It can be used by parties to the transfer to exchange information with each other about the exchange rate for the transfer, to ensure that the best rate can be agreed on The **FxRate** object will have the structure described below.

| Name         | Cardinality | Type   | Comment                                            |
|--------------|-------------|--------|----------------------------------------------------|
| sourceAmount | 1           | Amount | The amount of the transfer in the source currency. |
| targetAmount | 1           | Amount | The amount of the transfer in the target currency. |

## New data objects in the FXP API

The following data objects will be added to the FXP API to support currency conversion. Where data structures or elements are not defined in this document, they are to be found in Section 7 of **Ref 1** above.

### Charge

An FXP will be able to specify a charge which it proposes to levy on the currency conversion operation using a **Charge** object. The **Charge** object will have the structure described below.

| Name         | Cardinality | Type          | Comment                                                                           |
|--------------|-------------|---------------|-----------------------------------------------------------------------------------|
| chargeType   | 1           | String(1..32) | A description of the charge which is being levied.                                |
| sourceAmount | 0..1        | Amount        | The amount of the charge which is being levied, expressed in the source currency. |
| targetAmount | 0..1        | Amount        | The amount of the charge which is being levied, expressed in the target currency. |

### Conversion

A DFSP will be able to request a currency conversion, and an FX provider will be able to describe its involvement in a proposed transfer, using a **Conversion** object. The **Conversion** object will have the structure described below.

| Name                  | Cardinality | Type          | Comment                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|-----------------------|-------------|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| conversionId          | 1           | CorrelationId | An end-to-end identifier for the conversion request.                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| determiningTransferId | 0..1        | CorrelationId | The transaction ID of the transfer on whose success this currency conversion depends. If this is a bulk currency conversion which is not dependent on a transfer, then this field should be omitted.                                                                                                                                                                                                                                                                                  |
| counterPartyFsp       | 1           | FspId         | The ID of the FXP performing the conversion                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| amountType            | 1           | AmountType    | This is the AmountType for the base transaction, as described in Section 7.3.1 of **Ref 1** above. If it is set to **SEND**, then any charges levied by the FXP as part of the transaction will be deducted by the FXP from the **amount** shown for the **target** party in the conversion. If it is set to **RECEIVE**, then any charges levied by the FXP as part of the transaction will be added by the FXP to the **amount** shown for the **source** party in the conversion.  |
| initiatingFsp         | 1           | fspId         | The id of the participant who is requesting a currency conversion                                                                                                                                                                                                                                                                                                                                                                                                                     |
| sourceAmount          | 1           | FxMoney       | The amount to be converted, expressed in the source currency .                                                                                                                                                                                                                                                                                                                                                                                                                        |
| targetAmount          | 1           | FxMoney       | The converted amount, expressed in the target currency                                                                                                                                                                                                                                                                                                                                                                                                                                |
|                       | 1           | DateTime      | The end of the period for which the currency conversion is required to remain valid.                                                                                                                                                                                                                                                                                                                                                                                                  |
| charges               | 0..16       | Charge        | One or more charges which the FXP intends to levy as part of the currency conversion, or which the payee DFSP intends to add to the amount transferred.                                                                                                                                                                                                                                                                                                                               |
| extensionList         | 0..1        | ExtensionList | Optional extension list, specific to the deployment.                                                                                                                                                                                                                                                                                                                                                                                                                                  |

### FxMoney

The **FxMoney** object describes an amount which forms part of a currency conversion proposal. It is calqued on the **Money** object described in Section 7.4.10 of **Ref 1** above, but allows the amount to be optional to support quotations. The **FxMoney** object has the structure described below.

| Name     | Cardinality | Type     | Comment                                                                                                                                                                                                   |
|----------|-------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| currency | 1           | Currency | The currency of the amount, as described in Section 7.3.9 of **Ref 1** above                                                                                                                              |
| amount   | 0..1        | Amount   | The amount of the transfer, as described in Section 7.2.13 of **Ref 1** above. This amount will be present for the fixed element of the conversion, and absent for the variable element of the conversion |

## FXP API

The following resources will be supported by the FX API.

### services

The **services** resource allows a participant to obtain a list of the DFSPs in the scheme who provide a particular service (for the purposes of this document, a currency conversion service,) and to specify a particular currency for which they require a currency conversion.

This section describes the services that can be requested on the resource **/services**

##### GET /services/FXP/\<sourceCurrency\>/\<targetCurrency\>

The HTTP request **GET /services/FXP** is used to request information about the participants in a scheme who offer currency conversion services, and optionally to restrict the search to participants who offer currency conversion services in a particular currency corridor. The required corridor is specified by giving the ISO 4217 currency code for the source currency and the target currency in the *\<sourceCurrency\>* and *\<targetCurrency\>* fields: for instance **GET /services/FXP/KES/ZAR** is used to request a list of participants who support conversions from Kenyan shillings to South African rand.

A DFSP may request that the **services** call should return all the FXPs who will undertake conversion from, or to, a particular currency without specifying what the counter-currency should be. For instance, to return a list of all the FXPs who support conversion to ZAR:

**GET /services/FXP//ZAR**

To return a list of all the FXPs who support conversion from KES:

**GET /services/FXP/KES/**

**Note** that:

-   If no currency corridor is specified, then the service makes no guarantee that a specific FXP will in fact be able to perform any specific currency conversion.
-   The inclusion of a participant in the return list should not be taken to imply that the participant will agree to undertake any particular currency conversion.

##### PUT /services/FXP

The callback **PUT /services/FXP** is used to inform the requester about up to 16 participants in a scheme who offer currency conversion services. If no participants offer these services, the return object will be blank. If the call to the resource specified a currency corridor, this will not be indicated in the list of participants returned.

The data model for the callback is as follows:

| Name      | Cardinality | Type  | Comment                                                                     |
|-----------|-------------|-------|-----------------------------------------------------------------------------|
| providers | 0..16       | fspId | The FSP Id(s) of the participant(s) who offer currency conversion services. |

#### Requests

### fxQuotes

The **fxQuotes** resource will be used by a DFSP to request an FXP to confirm a proposed FX conversion.

The data model of the POST request will be a **conversion** object (see Section 7.3.2 above) with at least the following items present:

1.  : the ID to be used to identify the conversion request.
2.  : the FXP’s Fsp ID.
3.  : the fspId of the payer party.
4.  **source.currency**: the source currency to be used for the conversion.
5.  
6.  **target.currency**: the target currency to be used for the conversion.
7.  *Either* the **sourcemount.**, if the amount in the target currency is to be calculated, *or* the **targetmount.mount** if the amount in the source currency is to be calculated.

#### Requests

This section describes the services that can be requested on the resource **/fxQuote**

##### GET /fxQuotes/\<ID\>

The HTTP request **GET /fxQuotes/**\<ID\> is used to request information regarding a request for quotation for a currency conversion which the sender has previously issued. The \<ID\> in the URI should contain the **conversionRequestId** that was used to request the quotation.

##### POST /fxQuotes

The HTTP request **POST /fxQuotes** is used to ask an FXP to provide a quotation for a currency conversion. Its data model will be as follows:

| Name                | Cardinality | Type                      | Comment                                                               |
|---------------------|-------------|---------------------------|-----------------------------------------------------------------------|
| conversionRequestId | 1           | CorrelationId             | An end-to-end identifier for the quotation request.                   |
| conversionTerms     | 1           | [Conversion](#conversion) | The terms of the currency conversion for which a quotation is sought. |

#### Responses

This section describes the callbacks that are made by the server under the resource **/fxQuotes**.

##### PUT /fxQuotes/\<ID\>

The callback **PUT /fxQuotes/**\<ID\> is used to inform the requester about the outcome of a request for quotation for a currency conversion. The *\<ID\>* field in the URI should contain the **conversionRequestId** that was used when the quotation for the currency conversion was requested.

The data model for the callback is as follows:

| Name            | Cardinality | Type                      | Comment                                                                                          |
|-----------------|-------------|---------------------------|--------------------------------------------------------------------------------------------------|
| condition       | 0..1        | IlpCondition              | The ILP condition for the conversion.                                                            |
|                 |             |                           |                                                                                                  |
| conversionTerms | 1           | [Conversion](#conversion) | The terms under which the FXP will undertake the currency conversion proposed by the requester.  |

#### Error Callbacks

This section describes the error callbacks that are used by the server under the resource **/fxQuotes**.

##### PUT /fxQuotes/*\<ID\>*/error

If the FXP is unable to find or create a currency conversion execution, or another processing error occurs, the error callback **PUT /fxQuotes/\<***ID\>***/error** is used. The *\<ID\>* in the URI should contain the **conversionRequestId** that was used for the creation of the quotation request, or the *\<ID\>* that was used in the [**GET /fxQuotes/***\<ID\>*.](#_bookmark185) The data model for this callback is given below.

| Name             | Cardinality | Type             | Comment                                 |
|------------------|-------------|------------------|-----------------------------------------|
| errorInformation | 1           | ErrorInformation | The error code and category description |

### fxTransfers

The **fxTransfers** resource will allow a participant to ask an FXP to commit to a currency conversion in respect of a transfer. It is issued after the payee DFSP has confirmed that it proposes to commit a transfer, but before it returns the fulfilment to the payer DFSP.

#### Requests

This section describes the services that can be requested on the resource **/fxTransfers**

##### GET /fxTransfers/\<ID\>

The HTTP request **GET /fxTransfers/**\<ID\> is used to request information regarding a request for confirmation of a currency conversion which the sender has previously issued. The \<ID\> in the URI should contain the **commitRequestId** that was used to request the confirmation.

##### POST /fxTransfers

The HTTP request **POST /fxTransfers** is used to ask an FXP to confirm the execution of an agreed currency conversion. Its data model will be as follows:

| Name                  | Cardinality | Type          | Comment                                                                                                                                                                                                |
|-----------------------|-------------|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| commitRequestId       | 1           | CorrelationId | An end-to-end identifier for the confirmation request.                                                                                                                                                 |
| determiningTransferId | 0..1        | CorrelationId | The transaction ID of the transfer to which this currency conversion relates, if the conversion is part of a transfer. If the conversion is a bulk currency purchase, this field sfrehould be omitted. |
| requestingFsp         | 1           | FspId         | Identifier for the FSP who is requesting a currency conversion                                                                                                                                         |
| respondingFxp         | 1           | FspId         | Identifier for the FXP who is performing the currency conversion                                                                                                                                       |
| sourceAmount          | 1           | Money         | The amount being offered for conversion by the requesting FSP                                                                                                                                          |
| targetAmount          | 1           | Money         | The amount which the FXP is to credit to the requesting FSP in the target currency                                                                                                                     |
| condition             | 1           | llpCondition  | ILP condition received by the requesting FSP when the quote was approved.                                                                                                                              |
|                       |             |               |                                                                                                                                                                                                        |

#### Responses

This section describes the callbacks that are made by the server under the resource **/fxTransfers**.

##### PUT /fxTransfers/\<ID\>

The callback **PUT /fxTransfers/**\<ID\> is used to inform the requester about the outcome of a request for execution of a currency conversion. The *\<ID\>* field in the URI should contain the commitRequestId that was used when the execution of the currency conversion was requested.

The data model for the callback is as follows:

| Name               | Cardinality | Type          | Comment                                                                                                                            |
|--------------------|-------------|---------------|------------------------------------------------------------------------------------------------------------------------------------|
| fulfilment         | 0..1        | IlpFulfilment | The fulfilment of the condition specified for the currency conversion. Mandatory if the conversion has been executed successfully. |
| completedTimeStamp | 0..1        | DateTime      | Time and date when the conversion was executed.                                                                                    |
| conversionState    | 1           | TransferState | The current status of the conversion request.                                                                                      |
| extensionList      | 0..1        | ExtensionList | Optional extension list, specific to the deployment.                                                                               |

##### PATCH /fxTransfers/\<ID\>

The callback **PATCH /fxTransfers/**\<ID\> is used to inform the requester about the final determination by the switch of the transfer which determines the status of a request for execution of a currency conversion. The *\<ID\>* field in the URI should contain the commitRequestId that was used when the execution of the currency conversion was requested.

The data model for the callback is as follows:

| Name               | Cardinality | Type             | Comment                                                                                                                            |
|--------------------|-------------|------------------|------------------------------------------------------------------------------------------------------------------------------------|
| fulfilment         | 0..1        | IlpFulfilment    | The fulfilment of the condition specified for the currency conversion. Mandatory if the conversion has been executed successfully. |
| completedTimeStamp | 0..1        | DateTime         | Time and date when the conversion was executed.                                                                                    |
| conversionState    | 1           | TransactionState | The current status of the conversion request.                                                                                      |
| extensionList      | 0..1        | ExtensionList    | Optional extension list, specific to the deployment.                                                                               |

#### Error Callbacks

This section describes the error callbacks that are used by the server under the resource **/fxTransfers**.

##### PUT /fxTransfers/*\<ID\>*/error

If the FXP is unable to find or create a currency conversion execution, or another processing error occurs, the error callback **PUT /fxTransfers/\<***ID\>***/error** is used. The *\<ID\>* in the URI should contain the **commitRequestId** that was used for the creation of the execution request, or the *\<ID\>* that was used in the [**GET /fxTransfers/***\<ID\>*.](#_bookmark185) The data model for this callback is given below.

| Name             | Cardinality | Type             | Comment                                 |
|------------------|-------------|------------------|-----------------------------------------|
| errorInformation | 1           | ErrorInformation | The error code and category description |
