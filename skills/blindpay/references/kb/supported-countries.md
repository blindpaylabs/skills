# Supported countries

Every country BlindPay supports, by tier: standard, high-risk (Enhanced KYC required), and prohibited.

Source: https://blindpay.com/docs/kb/supported-countries

BlindPay supports customers and transactions across most countries. Which verification level a customer needs, and whether a country is supported at all, depends on where they are from. Country checks apply to a customer's `country`, `id_doc_country`, and (for businesses) each owner's `id_doc_country`, as well as bank account fields such as the SWIFT beneficiary/bank/intermediary country.

There are three tiers:

| Tier | What it means |
| --- | --- |
| Standard | KYC Standard (individuals) or KYB Standard (businesses) is available. Automated review for eligible individuals. |
| High-risk | Individuals must use KYC Enhanced. Manual review, up to 1 business day. Driver's license is not accepted as an ID document. |
| Prohibited | Not supported. Customer and bank account creation is blocked. |

**Note:**

High-risk status is a soft gate: it raises the KYC requirement, it does not block the country. Prohibited countries are a hard block.

## High-risk countries

Individuals from high-risk countries cannot use `kyc_type: standard`; creating one with `standard` is rejected. They must use `kyc_type: enhanced` instead, which requires the additional Enhanced KYC fields (source of funds, purpose of transactions) and goes through manual review.

- Manual review typically takes up to 1 business day.
- Driver's license (`DRIVERS`) is not accepted as an `id_doc_type` for a high-risk `id_doc_country`; use a passport or national ID card instead.
- Businesses are not gated by this list directly, but a business owner's `id_doc_country` is still checked against the prohibited-country list.

## Prohibited countries

Countries under sanctions or outside BlindPay's risk appetite are not supported at all. Creating a customer, adding a bank account, or setting a SWIFT beneficiary/bank/intermediary country that falls in this tier returns an error and the request is blocked. There is no override or manual exception path for a prohibited country.

## Country list

The table below lists every country and its current tier.

| Country | Code | Tier |
| --- | --- | --- |
| Åland Islands | AX | Standard |
| Albania | AL | Standard |
| Andorra | AD | Standard |
| Antarctica | AQ | Standard |
| Antigua and Barbuda | AG | Standard |
| Argentina | AR | Standard |
| Armenia | AM | Standard |
| Aruba | AW | Standard |
| Australia | AU | Standard |
| Austria | AT | Standard |
| Azerbaijan | AZ | Standard |
| Bahrain | BH | Standard |
| Bangladesh | BD | Standard |
| Belgium | BE | Standard |
| Belize | BZ | Standard |
| Bermuda | BM | Standard |
| Bhutan | BT | Standard |
| Bosnia and Herzegovina | BA | Standard |
| Bouvet Island | BV | Standard |
| Brazil | BR | Standard |
| Brunei Darussalam | BN | Standard |
| Bulgaria | BG | Standard |
| Cabo Verde | CV | Standard |
| Canada | CA | Standard |
| Cayman Islands | KY | Standard |
| Chile | CL | Standard |
| China | CN | Standard |
| Costa Rica | CR | Standard |
| Croatia | HR | Standard |
| Curaçao | CW | Standard |
| Cyprus | CY | Standard |
| Czechia | CZ | Standard |
| Denmark | DK | Standard |
| Dominican Republic | DO | Standard |
| Ecuador | EC | Standard |
| Estonia | EE | Standard |
| Falkland Islands [Malvinas] | FK | Standard |
| Faroe Islands | FO | Standard |
| Finland | FI | Standard |
| France | FR | Standard |
| French Guiana | GF | Standard |
| French Polynesia | PF | Standard |
| French Southern Territories | TF | Standard |
| Gambia | GM | Standard |
| Georgia | GE | Standard |
| Germany | DE | Standard |
| Gibraltar | GI | Standard |
| Greece | GR | Standard |
| Greenland | GL | Standard |
| Grenada | GD | Standard |
| Guadeloupe | GP | Standard |
| Guernsey | GG | Standard |
| Guyana | GY | Standard |
| Heard Island and McDonald Islands | HM | Standard |
| Holy See | VA | Standard |
| Honduras | HN | Standard |
| Hong Kong | HK | Standard |
| Hungary | HU | Standard |
| Iceland | IS | Standard |
| India | IN | Standard |
| Indonesia | ID | Standard |
| Ireland | IE | Standard |
| Isle of Man | IM | Standard |
| Israel | IL | Standard |
| Italy | IT | Standard |
| Japan | JP | Standard |
| Jersey | JE | Standard |
| Jordan | JO | Standard |
| Korea (the Republic of) | KR | Standard |
| Kuwait | KW | Standard |
| Latvia | LV | Standard |
| Lesotho | LS | Standard |
| Liechtenstein | LI | Standard |
| Lithuania | LT | Standard |
| Luxembourg | LU | Standard |
| Macao | MO | Standard |
| Malawi | MW | Standard |
| Malaysia | MY | Standard |
| Maldives | MV | Standard |
| Malta | MT | Standard |
| Marshall Islands | MH | Standard |
| Martinique | MQ | Standard |
| Mauritius | MU | Standard |
| Mayotte | YT | Standard |
| Mexico | MX | Standard |
| Micronesia (Federated States of) | FM | Standard |
| Mongolia | MN | Standard |
| Montenegro | ME | Standard |
| Montserrat | MS | Standard |
| Morocco | MA | Standard |
| Nauru | NR | Standard |
| Netherlands | NL | Standard |
| New Zealand | NZ | Standard |
| Niue | NU | Standard |
| Norway | NO | Standard |
| Oman | OM | Standard |
| Palau | PW | Standard |
| Papua New Guinea | PG | Standard |
| Paraguay | PY | Standard |
| Peru | PE | Standard |
| Philippines | PH | Standard |
| Pitcairn | PN | Standard |
| Poland | PL | Standard |
| Portugal | PT | Standard |
| Republic of North Macedonia | MK | Standard |
| Réunion | RE | Standard |
| Romania | RO | Standard |
| Saint Helena, Ascension and Tristan da Cunha | SH | Standard |
| Saint Kitts and Nevis | KN | Standard |
| Saint Lucia | LC | Standard |
| Saint Vincent and the Grenadines | VC | Standard |
| San Marino | SM | Standard |
| Sao Tome and Principe | ST | Standard |
| Serbia | RS | Standard |
| Seychelles | SC | Standard |
| Singapore | SG | Standard |
| Slovakia | SK | Standard |
| Slovenia | SI | Standard |
| South Georgia and the South Sandwich Islands | GS | Standard |
| Spain | ES | Standard |
| Sri Lanka | LK | Standard |
| Svalbard and Jan Mayen | SJ | Standard |
| Sweden | SE | Standard |
| Switzerland | CH | Standard |
| Taiwan | TW | Standard |
| Thailand | TH | Standard |
| Timor-Leste | TL | Standard |
| Turkey | TR | Standard |
| Turks and Caicos Islands | TC | Standard |
| Tuvalu | TV | Standard |
| United Arab Emirates | AE | Standard |
| United Kingdom of Great Britain and Northern Ireland | GB | Standard |
| United States Minor Outlying Islands | UM | Standard |
| United States of America | US | Standard |
| Uruguay | UY | Standard |
| Wallis and Futuna | WF | Standard |
| Algeria | DZ | High-risk |
| American Samoa | AS | High-risk |
| Angola | AO | High-risk |
| Anguilla | AI | High-risk |
| Bahamas | BS | High-risk |
| Barbados | BB | High-risk |
| Benin | BJ | High-risk |
| Bolivia (Plurinational State of) | BO | High-risk |
| Bonaire, Sint Eustatius and Saba | BQ | High-risk |
| Botswana | BW | High-risk |
| British Indian Ocean Territory | IO | High-risk |
| Burkina Faso | BF | High-risk |
| Cambodia | KH | High-risk |
| Cameroon | CM | High-risk |
| Chad | TD | High-risk |
| Christmas Island | CX | High-risk |
| Cocos (Keeling) Islands | CC | High-risk |
| Colombia | CO | High-risk |
| Comoros | KM | High-risk |
| Cook Islands | CK | High-risk |
| Djibouti | DJ | High-risk |
| Dominica | DM | High-risk |
| Egypt | EG | High-risk |
| El Salvador | SV | High-risk |
| Equatorial Guinea | GQ | High-risk |
| Eritrea | ER | High-risk |
| Eswatini | SZ | High-risk |
| Ethiopia | ET | High-risk |
| Fiji | FJ | High-risk |
| Gabon | GA | High-risk |
| Ghana | GH | High-risk |
| Guam | GU | High-risk |
| Guatemala | GT | High-risk |
| Haiti | HT | High-risk |
| Jamaica | JM | High-risk |
| Kenya | KE | High-risk |
| Kiribati | KI | High-risk |
| Madagascar | MG | High-risk |
| Mauritania | MR | High-risk |
| Moldova (the Republic of) | MD | High-risk |
| Monaco | MC | High-risk |
| Mozambique | MZ | High-risk |
| Namibia | NA | High-risk |
| Nepal | NP | High-risk |
| New Caledonia | NC | High-risk |
| Niger | NE | High-risk |
| Norfolk Island | NF | High-risk |
| Northern Mariana Islands | MP | High-risk |
| Panama | PA | High-risk |
| Rwanda | RW | High-risk |
| Saint Barthélemy | BL | High-risk |
| Saint Martin (French part) | MF | High-risk |
| Saint Pierre and Miquelon | PM | High-risk |
| Samoa | WS | High-risk |
| Senegal | SN | High-risk |
| Sierra Leone | SL | High-risk |
| Sint Maarten (Dutch part) | SX | High-risk |
| Solomon Islands | SB | High-risk |
| South Africa | ZA | High-risk |
| Suriname | SR | High-risk |
| Tanzania, United Republic of | TZ | High-risk |
| Togo | TG | High-risk |
| Tokelau | TK | High-risk |
| Tonga | TO | High-risk |
| Trinidad and Tobago | TT | High-risk |
| Tunisia | TN | High-risk |
| Uganda | UG | High-risk |
| Vanuatu | VU | High-risk |
| Viet Nam | VN | High-risk |
| Virgin Islands (British) | VG | High-risk |
| Virgin Islands (U.S.) | VI | High-risk |
| Western Sahara | EH | High-risk |
| Zambia | ZM | High-risk |
| Afghanistan | AF | Prohibited |
| Belarus | BY | Prohibited |
| Burundi | BI | Prohibited |
| Central African Republic | CF | Prohibited |
| Congo | CG | Prohibited |
| Congo (the Democratic Republic of the) | CD | Prohibited |
| Côte d'Ivoire | CI | Prohibited |
| Cuba | CU | Prohibited |
| Guinea | GN | Prohibited |
| Guinea-Bissau | GW | Prohibited |
| Iran (Islamic Republic of) | IR | Prohibited |
| Iraq | IQ | Prohibited |
| Kazakhstan | KZ | Prohibited |
| Korea (the Democratic People's Republic of) | KP | Prohibited |
| Kosovo | XK | Prohibited |
| Kyrgyzstan | KG | Prohibited |
| Lao People's Democratic Republic | LA | Prohibited |
| Lebanon | LB | Prohibited |
| Liberia | LR | Prohibited |
| Libya | LY | Prohibited |
| Mali | ML | Prohibited |
| Myanmar | MM | Prohibited |
| Nicaragua | NI | Prohibited |
| Nigeria | NG | Prohibited |
| Pakistan | PK | Prohibited |
| Palestine, State of | PS | Prohibited |
| Puerto Rico | PR | Prohibited |
| Qatar | QA | Prohibited |
| Russian Federation | RU | Prohibited |
| Saudi Arabia | SA | Prohibited |
| Somalia | SO | Prohibited |
| South Sudan | SS | Prohibited |
| Sudan | SD | Prohibited |
| Syrian Arab Republic | SY | Prohibited |
| Tajikistan | TJ | Prohibited |
| Turkmenistan | TM | Prohibited |
| Ukraine | UA | Prohibited |
| Uzbekistan | UZ | Prohibited |
| Venezuela (Bolivarian Republic of) | VE | Prohibited |
| Yemen | YE | Prohibited |
| Zimbabwe | ZW | Prohibited |

Ukraine's prohibited status reflects the sanctioned regions (Crimea, Donetsk, Luhansk, Zaporizhzhia, and Kherson).

**Warning:**

Country tiers are reviewed periodically as sanctions and risk assessments change, so a country's tier can move. Contact [support@blindpay.com](mailto:support@blindpay.com) if you need to confirm a country's current tier before launching there.

## What to build against

Because these tiers can change, avoid hardcoding the list into your own product logic. Instead:

- Let the customer creation call be the source of truth: submit the customer with the country and KYC type you intend, and handle the error response if the country requires a different tier.
- For individuals from high-risk countries, collect the Enhanced KYC fields up front. Enhanced KYC always goes through manual review; there is no fast-path back to standard's automated speed.

## Related

- [Payment methods](payment-methods.md): which bank rails are available in each country
- [KYC requirements](kyc.md): verification levels, required fields, and statuses referenced by country tier
- [Instances](../essentials/instances.md): sandbox vs. production behavior for customer creation and KYC
