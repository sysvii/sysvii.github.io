+++
title = "AWS SSO Deeeep Dive"
description = "A look at AWS SSO & how it looks from an end-user prospective"
date = 2021-04-11
updated = 2021-04-11

[taxonomies]
categories = ["tech"]
tags = ["aws", "sso", "identity"]
+++

<!--
 What to cover:
  - intro to AWS SSO
  - What the cli experience is like
  - What APIs are happening
-->

# What is even SSO?!

SSO is the fancy TLA for **S**ingle **S**ign **O**n, which is the great of idea
of having just having to login into one place & then authenticate into other
services when you want to. Some kind people have put great effort into the
[Wikipeia](https://en.wikipedia.org/wiki/Single_sign-on) page for it,
recommended to get more familiar with the idea.

Generally it is a very *Enterprise Plan* feature so us mere
mortals that cannot pay the usual USD$50/user/month (minimum 50 users) don't get to see it much.
Of course with the exception of the "Social" logins ("Sign in with Facebook/Google")
and AWS that provides it FOR FREE.

> **Note**: that none of these companies named sponsored this post

## The "_Lab_" Setup

 * Developer account for [Okta](https://developer.okta.com/)
 * Free tier [AWS Account](https://aws.amazon.com/)
 * Use [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html), SSO support is only in V2+

 * Setting up basic use of [Okta as the **Id**entity **P**rovider (IdP)](https://docs.aws.amazon.com/singlesignon/latest/userguide/okta-idp.html)

> I will not be going through how to setup this "_lab_", AWS & Okta have some
> [nice docs](https://docs.aws.amazon.com/singlesignon/latest/userguide/okta-idp.html) on how to


To start, the native way of getting credentials for a session are the following:

```
aws sso login <profile_name>
```

And from there lets see what has done.

## What has that done?

There is some new mess in your `~/.aws` folder.

```
> find ~/.aws -exec file {} \;
~/.aws: directory
~/.aws/config: ASCII text
~/.aws/cli: directory
~/.aws/cli/cache: directory
~/.aws/cli/cache/0000000000000000000000000000000000000000.json: JSON data
~/.aws/sso: directory
~/.aws/sso/cache: directory
~/.aws/sso/cache/botocore-client-id-ap-southeast-9.json: JSON data
~/.aws/sso/cache/1111111111111111111111111111111111111111.json: JSON data
```

And in the `~/.aws/config` there is now a profile

```
[profile RoleAccess-000000000000]
sso_start_url = https://d-0000000000.awsapps.com/start#/
sso_region = ap-southeast-9
sso_account_id = 000000000000
sso_role_name = RoleAccess
```

### Cache files?!

Those cache files are new and interesting. So what are in those?

#### CLI cache

`~/.aws/cli/cache/0000000000000000000000000000000000000000.json`

```json
{
  "ProviderType": "sso",
  "Credentials": {
    "AccessKeyId": "ASIA0000000000000000",
    "SecretAccessKey": " -- classic secret access key high entropy stuff here --",
    "SessionToken": " -- big old session token --",
    "Expiration": "2021-02-11T09:12:40Z"
  }
}
```

Taking these credentials out they actually work.

So this looks like how AWS CLI v2 handles your SSO sessions and just keeps them
to the side for you.

The file name looks to be a hash so guessing the name is pretty hard for a file
inclusion attack unlike the ye 'old `~/.aws/credentials`.

Though these look to be stable hashes per profile.

#### SSO File

`~/.aws/sso/cache/1111111111111111111111111111111111111111.json`

```json
{
  "startUrl": "https://d-0000000000.awsapps.com/start#/",
  "region": "ap-southeast-9",
  "accessToken": "eyJlbmMiOiJBMjU2R0NNIiwidGFnIjoiSm5udXRHU0tKTVVTb1RqRyIsImFsZyI6IkEyNTZHQ01LVyIsIml2IjoiazNiZXlaeEtDR0otc05VQyJ9.AYABeObaD6db3xtlf2VyQV4uZbcAHwABABBEYXRhUGxhbmVTZXNzaW9uAAlQZXJlZ3JpbmUAAQAHYXdzLWttcwBQYXJuOmF3czprbXM6YXAtc291dGhlYXN0LTI6NDEyMjAxMDI1ODU4OmtleS9kMTEwZTQ1ZC1mMGVhLTQ4NTYtYmFjNS0xMmMyYTMxM2Q2YWEAuAECAQB4Pcqs65qFZyXWdyHZA9hnHzXOhvyEOzDstw3xH4zNak0BY6qRWzxkq86qEu6rfKsXwQAAAH4wfAYJKoZIhvcNAQcGoG8wbQIBADBoBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDLXUMJLzS47DezlQSAIBEIA7lHcufjSiF9_gCd_IcRwk9NoCCnr9JmXvmWygjueAsEigiiXZ3WUF-K64_lSWTdaT9dC8rGDIh2-nEjoCAAAAAAwAABAAAAAAAAAAAAAAAAAAAFNQZGeo3pZnmaTvFFpgff____8AAAABAAAAAAAAAAAAAAABAAAAIAoOunYoPR2JIAc65uLD1QkKwC7dH1NzyRkba108s0-XEVJFBRIH1bPqAUpp7NaJyg.dzmoGCWoLlq-x4GZ.IHhQ2ffvTLAQQfffWXu9QvnEg1dir26R1HZ5gcNX1QcScFq15-AE_9ylR9djZyTv2qCk3xHRdx26oGhhK50m6DbryV2wy1OsEL_UpbAdt4x18upnSJsQXEcnug9mEbdP8P-XtXCzhCMT2yCahatqvsifKagyInM2vQATNZRc87r2pM4qy_z4h3rmzBpvXw6_lQ5AN7gZXN1urAGELHGanrHzkCX80msZskK1_RFIQbO0HADjfTnEeK1UmNXaXpFbq2oMI-NHIbkGsf8_XhY3AFX-EZfqS_MmgPAQAJ-fTF4P8vlJ4-UHQTX3onws_v3LWoqGA8Tlf-Vrf76mqUqN-ZUtnsUeP-0NVy2dDZu3z54xghta40Dbly9QVItaGzB9oSXJKn-NwiBdKxm6F0BhC9TShFghZ-k9P0JFCuHvl7lKlETTSH1tnQXlRvTlAwN7hagZWMqw91fZgg9v_Ujc6w.zj3d2TiDLW7ajD5NVf6P0A",
  "expiresAt": "2021-02-11T16:12:30Z"
}
```

Hmm a string starting with `ey`?! Is this a JWT?!

Header contains the following.

```json
{
  "enc": "A256GCM",
  "tag": "JnnutGSKJMUSoTjG",
  "alg": "A256GCMKW",
  "iv": "k3beyZxKCGJ-sNUC"
}
```

There is a few crypto stuff that has gone over my head, but from searching for
the algorithm (under `alg` key) `A256GCMKW` found a [helpful StackOverflow
thread](https://stackoverflow.com/questions/38122521/what-is-the-algorithm-string-for-agcm256-kw-in-java-cryptography-to-be-used-i).
The key points I got from it are

 * `AES256` encryption algorithm used with the following options,
 * `GCM` - to detonate it is in `GCM` mode
 * `KW` - to detonate that key wrapping is used

Does this help much? Not really.

Now for the body (decoded by the [CyberChef themselves](https://gchq.github.io/CyberChef/))

```text
...xæÚ.§[ß.e.erA^.e·......DataPlaneSession.	Peregrine....aws-kms.Parn:aws:kms:ap-southeast-2:412201025858:key/d110e45d-f0ea-4856-bac5-12c2a313d6aa.¸....x=Ê¬ë..g%Öw!Ù.Øg.5Î.ü.;0ì·
ñ..ÍjM.cª.[<d«Îª.î«|«.Á...~0|.	*.H.÷
... o0m...0h.	*.H.÷
...0..	`.H.e....0...µÔ0.óK.Ã{9PH....;.w.~4¢.Ø.t..ÂOM  §¯Òf^ù.Ê.îx....¢].ÖPRºâT.MÖ.õÐ¼¬`È.iÄ.........................Ô..ê7¥.æi;Å....À...@...........@.....®...GbH.Î¹¸°õBB°.·GÔÜòFFÚ×O,Ñq.$PQ }[> .¦.Íh. w9¨.%¨.Z±àfH..6}ûÓ,..}÷Ö^ïP¾q ÕØ«Û¤u..`pÕõAÄ...y.Or..]...¿j..|GEÜvê.¡.®t. Û¯%vÃ-N°BÔ¥°.·.uòêgH..\G'º.f.·Oðõí\,á.ÄöÈ&¡jÚ¯²'Êj...Í¯@.Íe.<î½©3.²Ï.w®lÁ¦õðêT9.Þàesuº°..±ÆjzÇÎ@.óI¬fÉ
Õ.HA³´..ã}9Äx.T.ÕÚ^.[«j. ÑÈnA¬.Åácp.\F_©#&.ð...Ó..ü¾RxPt._z'ÂË÷-j*....õk.¾¦©J.eKg±G.ÐÕrÙÐÙ»|ùã.!µ®4
¹rõ.Hµ¡³.Ú.\.§7..t¬fè]../SJ.`..=?BE
áï.¹J.DÓH}m..åFôå..{.¨.XÊ°÷WÙ..oR7:Ä
```
Um... wat?! This looks very cyber encrypted but also is that a ARN in there?
And what is this `DataPlaneSession`?!

That KMS key id `arn:aws:kms:ap-southeast-2:412201025858:key/d110e45d-f0ea-4856-bac5-12c2a313d6aa` is a bit strange.
The account id is not one of _mine_, so it must be an AWS managed account. So
lets try some basic KMS operations on the key to see if there is a permissive
key policy.

```bash
> aws kms describe-key --region ap-southeast-2 --key-id arn:aws:kms:ap-southeast-2:412201025858:key/d110e45d-f0ea-4856-bac5-12c2a313d6aa

An error occurred (AccessDeniedException) when calling the DescribeKey operation:

> aws kms encrypt --region ap-southeast-2 --key-id arn:aws:kms:ap-southeast-2:412201025858:key/d110e45d-f0ea-4856-bac5-12c2a313d6aa --plaintext "$(echo lol | base64 -)"

An error occurred (AccessDeniedException) when calling the Encrypt operation:

> aws kms decrypt --region ap-southeast-2 --key-id arn:aws:kms:ap-southeast-2:412201025858:key/d110e45d-f0ea-4856-bac5-12c2a313d6aa --ciphertext-blob "$(base64 jwt_body)"

An error occurred (InvalidCiphertextException) when calling the Decrypt operation:
```

Ok, so that is mostly a no from what I know about KMS keys. Though that `InvalidCiphertextException` might be an [oracle vector](https://en.wikipedia.org/wiki/Oracle_attack).

I do wonder if it is the same KMS key for everyone that has deployed their SSO
into that region or if it is per customer.

The former might allow a path for token reuse between SSO instances, while the
latter would mean a $1/month/per customer instance fee for the SSO team that they
are kindly not passing onto the customer (please thank your local SSO team
member today).

If I ever get around to make more SSO instances I'll update this post with
those findings.

#### Signature section

There isn't really anything we can get from this section as the algorithm is
symmetric (AES) so we are not gonna get anything from it.

### Access Token

Jumping back a bit, but that `accessToken` in `~/.aws/sso/cache/1111111111111111111111111111111111111111.json` file is really interesting.
It is used with the AWS SSO API to vend out access keys into accounts.

It is essentially a replacement for your login credentials to your IdP, MFA and
all but with a time boxed existence.
From this token you can iterate all your accounts you have access to, the
roles in each of those accounts and then generate access keys for them.

The flow goes:

 * `sso:ListAccounts` - takes the access token. Returns a list of accounts that
     the token can access
 * `sso:ListAccountRoles` - takes the access token, and account id. Returns a
     list of roles the token can use in said account
 * `sso:GetRoleCredentials` - takes the access token, account id and role name.
     Returns working access keys for the role in said account.

For details about these actions have a look at the [Boto3 docs for SSO service](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sso.html)

These actions are recorded in CloudTrail. To be honest I am surprised as these
are at all since they are using Access Tokens instead of `STS` access keys.

Here is an example event:

```json
{
    "eventVersion": "1.08",
    "userIdentity": {
        "type": "Unknown",
        "principalId": "aaaaaaaaaa-bbbbbbbb-cccc-dddd-eeee-ffffffffffff",
        "accountId": "000000000000",
        "userName": "user@example.com"
    },
    "eventTime": "2021-03-28T00:26:05Z",
    "eventSource": "sso.amazonaws.com",
    "eventName": "ListAccountRoles",
    "awsRegion": "ap-southeast-2",
    "sourceIPAddress": "8.8.8.8",
    "userAgent": "Boto3/1.17.5 Python/3.9.2 Linux/5 Botocore/1.20.5",
    "requestParameters": null,
    "responseElements": null,
    "requestID": "7e0ffa90-e6cc-497a-982e-42ef7db5b415",
    "eventID": "b02878f9-a936-4027-9338-16fdba509113",
    "readOnly": true,
    "eventType": "AwsServiceEvent",
    "managementEvent": true,
    "eventCategory": "Management",
    "recipientAccountId": "000000000000",
    "serviceEventDetails": {
        "account_id": "000000000000"
    }
}
```

# Self-plug

From working with SSO I hobbled together a couple of scripts to automate the
setup of some of the annoying parts of AWS SSO, namely `aws sso login`
ssuucckks.

You get shoved to a browser every time when you got perfectly good cached
credentials on disk! So these scripts [drbarnz/aws-sso-tools](https://github.com/drbarnz/aws-sso-tools)
handle most of the time when you have to use that command.

 * Initial login
 * Switching profiles (account/role)
 * Generating profiles in `~/.aws/config`

# Take Aways

 * SSO is cool if you want short lived credentials locally available
 * `~/.aws/credentials` are out `~/.aws/cli/${HASH}.json` are in for access
     keys
 * `~/.aws/sso/cache/${HAS}.json`'s `accessToken` is same as login credentials
     but time boxed
 * SSO sessions do not make a mess of environment variables

## Extra Reading

 * [AWS SSO's docs on SSO](https://docs.aws.amazon.com/singlesignon/latest/userguide/understanding-key-concepts.html)
 * [Boto3 docs for SSO service](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sso.html)
 * [AWS SSO tooling](https://github.com/drbarnz/aws-sso-tools)
