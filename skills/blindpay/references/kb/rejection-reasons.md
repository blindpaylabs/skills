# Rejection reasons

Reason codes and messages returned when an application or document is rejected during verification.

Source: https://blindpay.com/docs/kb/rejection-reasons

## Summary

When an application or document is rejected during verification, BlindPay returns a reason code and message explaining the outcome. The table below lists every code, from document-quality issues (corners cut off, low resolution, photocopies) to compliance and commercial decisions. Most document rejections can be resolved by re-uploading a clear, full, in-date original.

## Reason codes

| # | Reason Code | Reason Message |
| --- | --- | --- |
| 1 | Address Mismatch | The account has been rejected due to a mismatch between the address provided in the [source of funds](source-of-funds.md) and the address on the [proof of address](proof-of-address.md) document. Address mismatches are not permitted, both documents must show the same, matching address. Please resubmit with documents reflecting a consistent address. |
| 2 | Alternative ID Document Required | We were unable to verify the authenticity of the document. Please resubmit the onboarding using a different identity document. |
| 3 | Business activity | The business industry classification (NAICS, CNAE or similar) listed in the articles of incorporation does not match the company's stated business activity. |
| 4 | Commercial | Commercial Reasons. |
| 5 | Compliance Verification | This decision is based on non-compliance with the criteria established in our Internal Compliance Policies and risk assessment guidelines. For reasons of confidentiality and security, we do not provide specific details regarding our internal approval criteria. |
| 6 | Document corners cut off | We were unable to accept the Identity Document because some edges or corners were cut off in the photo. Please re-upload a clear, full-page photo, ensuring that all four corners of the document are completely visible within the frame. |
| 7 | Document not readable | We were unable to verify the Identity Document because the document image provided is blurry, has glare, or is too low resolution. Please upload a new, clear photo, ensuring that all four corners of the document are visible and all text is easy to read. |
| 8 | Document photo with low quality | We were unable to verify the Identity Document because the document image provided is blurry, has glare, or is too low resolution. Please upload a new, clear photo, ensuring that all four corners of the document are visible and all text is easy to read. |
| 9 | Document Sides | Both sides of the identity document are required. |
| 10 | Duplicated | Duplicated Account. |
| 11 | Expired | The documentation provided is expired. Please submit a valid document. |
| 12 | Incorrect Document Order | We were unable to verify the identity document because the images were uploaded to the wrong fields (the back of the ID was uploaded in the front field, or vice versa). Please restart the onboarding process and resubmit the document, ensuring that the front side (with the photo) and the back side are uploaded into their respective fields. |
| 13 | Incorrect Proof of Address Document (POA) | The Proof of Address document provided does not meet our [compliance requirements](proof-of-address.md). Please ensure it is less than 90 days old, shows the full page, contains an emission date, and is issued under the user/entity name. Additionally, we do not accept screenshots. Please make sure to submit the official PDF document. |
| 14 | Incorrect Proof of Address/Source of Funds | We were unable to verify the Proof of Address/Source of Funds because the document provided is blurry, has glare, or is too low resolution. Please note that we do not accept screenshots as documents. Kindly provide the original PDF file for the Proof of Address and/or Source of Funds. |
| 15 | Incorrect Source of Funds | Kindly provide one of the following [documents](source-of-funds.md): bank statements for the last 3 full months; audited financial statements for the last year; or tax returns or annual tax filings for the last year. |
| 16 | Invalid Tax ID | The application was rejected because the customer does not have a valid tax identification number issued by their country of residence or incorporation. |
| 17 | Live Selfie | The selfie is not live. Please retake it in real time. |
| 18 | Low quality selfie | We couldn't verify the selfie because it was blurry, dark, or out of focus. Please restart the onboarding process and try again with a clearer, well-lit selfie. |
| 19 | Multiple entity documents submitted | Please start a new onboarding process and submit only one business entity. Submitting different entities for the same onboarding process causes a systemic reading error. |
| 20 | Multiple identity documents submitted | Please start a new onboarding process and submit only one type of identification document. Submitting different types of ID in the same onboarding process causes a systemic reading error. |
| 21 | Nested | Nesting occurs when a customer uses their account to send or receive funds on behalf of a third party that BlindPay has not onboarded, screened, or approved. For more details, see [Nested Payments](nested-payments.md). |
| 22 | No Response to RFI | Application rejected due to customer failure to respond to Enhanced Due Diligence (EDD) / [RFI](information-requests.md) requests. |
| 23 | Photocopy | New identification document is required. The attached ID is a scan, photocopy, picture of a screen, or a cropped image. Please upload a fresh photo of the original document taken directly from the webcam or phone camera. |
| 24 | Prohibited Activities | For more information, please check [Prohibited Activities](prohibited-activities.md). |
| 25 | Regulatory Rejections | Due to recent regulatory updates under the PSAV (VASP) framework, BlindPay is no longer accepting companies engaged in financial or intermediation activities. |
| 26 | Reversal Payments | Reversal Payments. |
| 27 | Test Onboarding | Test onboarding with non-genuine documentation. |
| 28 | Wrong document type selected | Please make sure to select the exact type of documentation being sent. |
| 29 | Your documents need some adjustments | We found several documents that don't meet our requirements, so we couldn't complete your onboarding. Check the details in [KYB documents](kyb-documents.md) and resubmit your documents following the guidelines. |

## Related

- [KYC Basics — photo tips](kyc-basics.md#photo-tips) · [Proof of Address](proof-of-address.md)
- [Requests for Information](information-requests.md)
- [KYB documents](kyb-documents.md) · [Source of funds](source-of-funds.md) · [Prohibited activities](prohibited-activities.md)
