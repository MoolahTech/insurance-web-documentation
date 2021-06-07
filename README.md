# Documentation for the web SDK for insurance

The Insurance SDK can be used to offer various insurance products to customers by partners. The data collection, handling, communication with the insurance provider, payments and reconciliation is handled by Savvy. Savvy also generates receipts, which are used as temporary policies until the final policy is issued, which are sent over email / whatsapp to the end customer. Savvy *does not* handle claim processing; for this we provide dedicated communication channels to the insurance company (email & phone number) for the end customer to use.

Savvy Environments:

Sandbox
* **API**: https://www.thesavvyapp.in/api/
* **Web SDK**: https://insurance.thesavvyapp.in

Production
* **API**: https://api.savvyapp.in/api/
* **Web SDK**: https://insurance.savvyapp.in

To use the SDKs, you have to first get access to your access key and secret key. Request your SPOC for these credentials. Staging credentials will be given first, and then production credentials will be issued once a round of testing has been performed.
**The secret key must be kept secret**. Please make sure this key is not on the phone, or anywhere in your database or permanent storage. It must be kept in your live environment as a env. variable or use other similar key storage mechanisms like AWS KMS. Leakage of your secret key could compromise all your users and could lead to very bad things.

## User creation

Every request from the SDK to the our API must be authenticated using your access key (which identifies the partner making the request) and an IDENTITY_TOKEN (which identifies the user making the request). A user must be created via a server-to-server call using your access and secret key. In return, an expiring token is passed back. Please store this token SAFELY, preferably in the android keystore or other similar storage mechanism and pass it to the SDK.

**User create call**

**POST** ($BASE_URL)**/partners/users**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-PARTNER-SECRET-KEY | string | xyzab |

Params (root key must be user: `user: { phone_number.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| phone_number | string | 9876543210 OR +919876543210 |
| email | string | test@example.com |
| first_name | string | Foo |
| last_name | string | Bar |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-USER-IDENTITY-TOKEN | string | abcd.efg.hijk |
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-hijk-xyz |

The identity token expires every 24 hours, so please make sure to have a refresh mechanism built in.

**Token refresh call**

**POST** ($BASE_URL)**/partners/users/new_token**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-PARTNER-SECRET-KEY | string | xyzab |

Params (root key must be user: `user: { uuid.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-rf-rrrr |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-USER-IDENTITY-TOKEN | string | abcd.efg.hijk |
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-hijk-xyz |

## Calling the web SDK

For calling out to the SDK, hit the previously mentioned URL with the following arguments:
Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-USER-IDENTITY-TOKEN | string | xyzab |

Params:
| name | type | example | mandatory |
| ---- | ---- | ------- | :-------: |
| partner_transaction_id | string, generated by you, identifies the transaction | XYZ123 | true |
| product_type | string | "job_loss", "covid" (complete list provided separately) | true |
| provider | string | "chola", "icici_lomabrd" (complete list provided separately) | true |
| coverage_amount | double | 50000 | false, but if passed in, make sure it is a valid value according to the table shared with you. Please note the user may change the coverage amount before transacting. This will be shared with you in the callbacks |

## Callbacks

The insurance transaction can be in various states, which are communicated with you via callbacks.
1. New insurance transaction begun
2. Insurance transaction completed
3. Insurance transaction expired / abandoned

Each callback is sent with a hash string. This hash string is a pipe-joined string of all the params sent, which is then hashed using HMAC with SHA256 using your secret key. You **must** verify this hash at your end, otherwise attackers might simply be able to spoof requests to your open endpoints.

**New insurance transaction begun**
Params sent:
```ruby
      {
        transaction_type: 'insurance_purchase',
        transaction_id: <YOUR_ID>,
        status_is: 'pending',
        status_was: nil,
        coverage_amount: 1000,
        premium: 100,
        end_date: "31/12/2099",
        created_at: "01/01/2021",
        hash: <HASH STRING>
      }
```

The hash string is generated as follows:
```ruby
hash_string = "transaction_type|transaction_id|status_is|status_was|coverage_amount|premium|end_date|created_at"
hash = HMAC('sha256', hash_string, secret_key)
```

**Insurance transaction completed**
Params sent:
```ruby
      {
        transaction_type: 'insurance_purchase',
        transaction_id: <YOUR_ID>,
        status_is: 'completed',
        status_was: <PREVIOUS STATUS>,
        coverage_amount: 1000,
        premium: 100,
        end_date: "31/12/2099",
        created_at: "01/01/2021",
        hash: <HASH STRING>
      }
```
The possible statuses are: `'pending', 'completed', 'error'`

**Insurance transaction error**
Params sent:
```ruby
      {
        transaction_type: 'insurance_purchase',
        transaction_id: <YOUR_ID>,
        status_is: 'error',
        status_was: <PREVIOUS STATUS>,
        coverage_amount: 1000,
        premium: 100,
        end_date: "31/12/2099",
        created_at: "01/01/2021",
        hash: <HASH STRING>
      }
```
The possible statuses are: `'pending', 'completed', 'error'`
