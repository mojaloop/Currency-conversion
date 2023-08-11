# Currency-conversion

Contains material relating to the currency conversion workstream

The objectives of the currency conversion workstream are to extend the Mojaloop API and the associated core Mojaloop code to support payments where currency conversion is required. In particular, it should support the following features:

- Either bulk or Payment versus Payment should be supported
- Currency conversion will be initiated in a dialogue between a DFSP and a Foreign Exchange Provider
- Currency conversion may be initiated by the debit or the credit party to a payment.
- Currency conversion should support conversion via a reference currency.
- Currency conversion functions should be configurably automated in Payment Manager. Configuration options should include:
  - Whether currency conversion is to be undertaken by the debtor or the creditor party
- An example Payment Manager acting as a Foreign Exchange Provider. Configuration options should include:
  - Currencies supported
  - An exchange rate for each currency to a reference currency
  - Fees to be charged
