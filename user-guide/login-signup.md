# Login & Signup

## Data and Access Context

Our Gen3 instances are configured in such that

1. Dataset that are public (i.e. no login nor identification is required)
2. Dataset that are available only for login user 
   1. You will need to identify against your home institution
   2. Your home institution must participate as one of Identity Provider
3. Dataset that are required applying Consent Petition or Program Enrollment Signup

## 1. Public

- You can access dataset without needing any login when visiting Gen3 instance.
- If you do not see any public dataset then Gen3 instance offer no public dataset.

## 2. Login Only

- Generally our Gen3 instances are integrated with CILogon for identity and access federation.

#### UMCCR Data Commons

- Goto https://gen3.cloud.dev.umccr.org/login
- Click "**CILogon Login**" button.
- Follow screens to select your home institution account and authenticate.
- Once authenticated, you will be redirected to UMCCR Data Commons homepage.
- You should be able to read-only access to
  - Program: `demo`
  - Project: `super`
  - https://gen3.cloud.dev.umccr.org/demo-super

## 3. Consent Petition

_or Program Enrollment Signup_

- Goto UMCCR Data Commons Consent Petition URL at:  
> https://registry-test.biocommons.org.au/registry/co_petitions/start/coef:22
- Follow screens to select your home institution account and authenticate.
- At "Self Signup With Approval" screen, fill up mandatory fields (annotated with asterisk * character) and click "Submit" will give you [**_Thank You for Registering_**](img/cilogon_signup_success_register.png) message.
- You Signup application is now pending approval and Admins are notified of new enrollment application.
- Momentarily, you should receive email verification. Please follow up to verification link. You will be (once again) asked to authenticate against your home institution.
- You should then see [**_Thank You for Verifying You Email_**](img/cilogon_signup_success_email_verify.png) message.
- Once Admin approved your consent petition and, you have verified your email, enrollment is completed.

#### Dataset Consent

- At this moment, applying to a particular dataset consent is **manual**. You will need to **contact Admin**(s) with
  - Your (gen3) username -- typically `firstname.lastname`
  - A particular **program** or **project** or **dataset**, if known
  - Admin will then update authorization (AuthZ)


---

> NOTE: _At the moment, we are still experimenting on this. See [discussion in workshop](https://github.com/umccr/gen3-doc/blob/main/workshop/2021-05-21.md)._


Mapping aforementioned stages to [GA4GH Standard Passport scheme][1]

- After Login against your home institution, you bear `AffiliationAndRole` scope.
- After Consent Petition (or Program Enrollment Signup), you bear `AcceptedTermsAndPolicies` and/or `ResearcherStatus` scope.
- After Dataset Consent application done, you bear `ControlledAccessGrants` and/or all others scopes.


[1]: https://github.com/ga4gh-duri/ga4gh-duri.github.io/blob/master/researcher_ids/ga4gh_passport_v1.md#ga4gh-standard-passport-visa-type-definitions
