# authbypass-authentication-flaws

If the page opens on login / SSO, follow `dig-scope` §4.1.1: find the business surface; do not run dictionaries / CAPTCHA / unlimited password attempts from the start of this file.  
Probe sessions, reset, rebinding, ticket exchange, and 2FA using this file + `dig-scope` §4.2.2; do not skip the whole category because of §4.1.1. This is the minimum for each site, not permission to test only these cases.  
Bare default middleware endpoints can be obvious. Slider / code-sending / password attempts that never reach an account → switch to the authentication chain; do not stop halfway. Whether to write a report is determined only by `vuln-report-format.md`.
The 20 English dictionary/CAPTCHA methods and reset matrix have been cut; search headings for short-table pointers. See `http-host-header-test.md` for Host poisoning during password reset. See `csrf-test.md` §18 for QR-code login CSRF.

# Authentication Bypass

### Change password while logged out

Recognize: The final step for change password / first-time password setup / reset is reachable while logged out. The body commonly has an old-password or verification-code field (`old_pwd` / `sms_code`, etc.) + new password + identity ID. The official page may say SMS/old password is required, while the endpoint still accepts an empty string.

Test: While logged out, blank or omit the old password/code and use a new password that meets complexity requirements. Control: an incorrect code should be blocked; a nonexistent identity ID should return no result. If retrying the same password reports “same as historical password,” it has already been written to the database; that is not an empty success. Revert it immediately if it worked; if you cannot revert it, stop. Do not try another person's account or log into it.

Success: `Success` while logged out changed someone else's password.

False positives: Returns 0 without writing to the database; real old password/SMS is required; `Success` but login uses another IdP and the password did not carry over.

### IDaaS unused security questions

Recognize: IDaaS password recovery. Anonymous username submission issues a JWT, and `scope` is `_` (not reset). Security-question bank IDs can be enumerated; writing an ID already bound to that account returns 409. A common IDaaS skin uses path `forget_password/v2/sq`; test other skins with the same pattern.

Test:

1. While logged out, POST sq with body `{"username":"admin"}` (or an account first confirmed through an account-existence endpoint).
2. With the token, GET question and note the already-bound IDs.
3. POST `update_question` with answers for **unused** IDs (do not force-write already-bound IDs).
4. Submit your own three questions to `verifyquestions?type=RESET_PASSWORD` and obtain `self.password.expired.reset`.
5. `set_password`: if it works, change it back immediately; if you cannot, stop at the response and do not leave the administrator password changed. If policy rejects it and the security-question data was not written → partial only; do not count it as a full compromise.

Success: Security-question answers are written to someone else's account (rebind), or the token actually changes someone else's password.

False positives: The sq token gets `scope is not expected` at `set_password`; only already-used questions can be written and return 409; wrong answers at verify; policy rejects the operation and no security-question data was written. A miss on one site does not remove this row from the short table.

### Unauthenticated training-account binding endpoint (short table has a pointer)

Recognize: A training/college H5 gateway puts account binding and query RPCs under an unauthenticated client prefix (such as `/spapi/v2/client/`). The body accepts a C-side UID (`mtUserId`) + merchant account (`bAccountId`); no Cookie is needed.

Test (while logged out):

1. POST bind: replace uid with someone else's and replace the merchant account with one you make up.
2. POST querybind: send only uid or only the merchant account.

Count as success: Returns the other party's phone number, or binding succeeds and changes the binding.

False positives: Only returns unauthenticated/invalid parameters; uid is always unbound to a phone and cannot be written. A miss on one site does not remove this row from the short table. Do not store actual secret values or a particular phone number.

### Federated login accepts only uin (short table has a pointer)

Recognize: The token endpoint for a cloud event / hackathon / partner portal accepts only `provider` + cloud account `uin` (or similar uid) in the body, without validating OAuth `code` / `ticket` / `id_token`. Public lists or by-code lookup can reveal another person's UIN.

Test (while logged out):

1. Copy a `uin` from a public partner/user list or query by-code first.
2. `POST` the token endpoint with only `provider` + the other person's `uin`.
3. Use the JWT on `me` / user information.

Count as success: `me` is the **other person's** UIN and you can act as that account. A guest/empty account alone does not count.

False positives: Requires a real OAuth code; unregistered uin is rejected; a ticket is issued but `me` is still your own account. A miss on one site does not remove this row from the short table. Do not store actual secret values or a particular JWT.

### Event lab login does not validate the federated ticket (short table has a pointer)

Recognize: Event-page login accepts federated `openid`+`acctype`+`access_token`, and the server **does not validate** this ticket with the federation provider. Control gate: the same parameters with `acctype=wx` or empty `access_token` fail login; `acctype=qq`/`qc`/`pt` with a fake token still issues a ticket. This is not the generic empty-password/omit trick, nor the cloud-event case that accepts only uin. Frontend `fx_act_helper.js` / Milo login helper are common skins; test even if that name is absent.

Test (while logged out):

1. POST `/api/login` (follow the event prefix used by the page; do not assume one path), set `access_token` to a fake value, `acctype` to qq (try qc/pt separately), and `openid` to an enumerable account.
2. Use the JWT on `getRoles` (by game region) / `bindRole` or a similar role endpoint. Control: without the header it should require login; wx/empty ticket should fail.
3. Do not treat an empty role list as no issue: change region or try a neighboring account; follow the bind endpoint until binding works. Revert it if possible.

Count as success: The ticket contains the other person's openid, the role list shows a role name/`charac_no` under their account, or you can bind it.

False positives: Only issues an empty account and every role list is empty, with no way to bind a game character; a real federation ticket is required. Do not store actual secret values or a particular JWT. A miss on one site does not remove this row from the short table.

### Session ticket in a deep link in operations configuration (short table has a pointer)

Recognize: Unauthenticated homepage/operations-configuration JSON contains a banner redirect URL (skipPath / jumpUrl / schema) whose query includes a session-capable string such as `token` / `apptoken` / `access_token`. Control gate: calling `me` without that string should require login / report expiration.

Test (while logged out):

1. Copy the full redirect URL from the configuration and put its query ticket in the Token / Authorization header.
2. Call me / user info / wallet.
3. Control: the same endpoint without that string should be expired.

Count as success: `me` returns someone else's phone/character and the session can act as that account.

False positives: Ticket is expired or placeholder and cannot log in; configuration contains only operations copy and no ticket. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Signature-calculation leak in 302 after failed verification (short table has a pointer)

Recognize: A payment/open-platform gateway envelope has `sign_data`/`sign` + merchant ID. With a fake signature, **GET / DELETE / OPTIONS / HEAD may all return 302** and the Location or `msg` contains the valid signature the server just calculated (such as `calculateSign is:`). Some gateways return only HTML/ILLEGAL_SIGN for GET and leak only after changing METHOD; others **leak the signature in a 302 on GET itself**. Do not test only DELETE/OPTIONS.

Test (while logged out):

1. Send a fake signature to a sign-contract/order endpoint; **check Location on GET first** for `calculateSign`.
2. If GET does not include the signature, switch to DELETE/OPTIONS/HEAD and decode/copy it from the Location URL.
3. Put it back in `sign_data` and call query/order. Control: fake signature still fails; real signature returns another merchant's order.

Count as success: Passing verification exposes **another merchant's** unpublished order/account body (amount, close reason, card type).

False positives: Every METHOD reports signature failure and none calculates a signature; copied signature is still rejected. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Identity-provider proxy echoes token in an error (short table has a pointer)

Recognize: A business endpoint calls an identity provider for the frontend. Three common skins: ① mini-program-code/redirect-code endpoint lets you choose `path`/`page`/`pagepath`, and for an invalid page the message **echoes `access_token=` verbatim**; ② QQ/OAuth token-exchange endpoint concatenates `client_secret` into the upstream URL, changing `redirect_uri` to an external domain triggers downstream HTTP 5xx, and the error URL exposes `client_secret=` to the frontend; ③ unauthenticated business endpoint `GET /api/business/access-token` (or similar ticket path) directly returns a ticket with 200, without waiting for an error echo. A fake code sent to a same-domain callback often returns only a business error code without the key; **do not stop**. If the inventory has a ticket endpoint, do not test only the code-signing endpoint.

Test (while logged out):

1. If the inventory has a ticket endpoint, GET it first; 200 with `data` equal to `access_token` means the value is already exposed, with no need to try an invalid pagepath.
2. If no ticket endpoint exists: set path to an external site or clearly invalid page; for token exchange, first use a fake code + same-domain `redirect_uri` as a control.
3. Then change the token-exchange `redirect_uri` to an external domain and copy `access_token=` or `client_secret=` from the error URL/message.
4. Use the identity ticket on `Identity Provider API/cgi-bin/account/getaccountbasicinfo`, then `/wxa/generatescheme`; use QQ AppSecret with an authorization code to exchange for the app's access_token / test this site's token exchange.

Count as success: Retrieve the official account's owner/appid, issue an official production redirect code, or use the complete AppSecret to exchange for that app's access_token.

False positives: Error contains only ErrCode and no token/secret; fake code to same-domain callback does not expose a key; token is invalid on the identity API; secret cannot exchange a ticket. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Enablement-page RPC returns signKey (short table has a pointer)

Recognize: Payment UISDK/enablement page. While logged out, call config or thrift endpoint with only an enumerable `appId`. A dead application returns `APP_NOT_FOUND` with no key; a live one returns complete `signKey`/`signMethod`. This is not a key hardcoded in docs/webpack (see the next section).

Test (while logged out):

1. Change `appId`. Control: dead account should return no key.
2. Copy a live account's `signKey`, then sign a request using the returned `signMethod` (often MD5; sort parameters lexicographically and append `&key=`).
3. Call payment query. Control: fake key returns `SIGN_ERROR`; real key passes.
4. Do not stop after query passes. Call settlement-card change `change/card`: vary the branch number to distinguish `changeId`; unchanged value reported as not modified is evidence of a successful write. SUCCESS with `changeId=null` / unchanged fee rate does not count. Revert immediately if it works.

Count as success: Complete AppSecret and real key passes production verification; or query exposes another merchant's ID/card/pending order; or creates a virtual-store poiId; or changes another merchant's settlement card/bank (nonempty `changeId`).

False positives: Returned placeholder key cannot pass verification; both dead and live IDs return `APP_NOT_FOUND` with no key. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### AppSecret hardcoded in debugging documentation (short table has a pointer)

Recognize: Documentation / Demo / official package / integration HTML / old npm package has a **complete** AppSecret hardcoded (appId+appSecret, signKey, merchant pfx, SDK key), not a `your-secret` / `YOUR_ACCESS_KEY` placeholder. **If the wiki masks the key with asterisks but the same official zip / Demo / HTML example still has it in plaintext, keep going. Do not treat an old npm release as withdrawn.** Common skins: debugging guide `config.js`, GitHub `properties` linked from FAQ, operation screenshot PNG in docs, SDK zip `prod/config.properties` / Demo Java `baseInfo`, integration HTML `MSDKConfig.json`, public npm `index.js`. There is a nearby token-exchange endpoint or production endpoint that can be signed as documented. The algorithms below are test directions, not tied to a single product.

Test (while logged out):

1. Copy appId/accessKey and appSecret/secretKey from documentation, linked GitHub, **or an operation screenshot in the docs** (put real values only in the report, not this repository). OCR/look at the screenshot; do not stop at config.js.
1b. Enterprise IM/open directory: if `user/get` or `user/list` after ticket exchange has empty `mobile`, **do not stop**; call `linkedcorp/user/get` (body `userid`). Control: directory endpoint has no phone; linked-enterprise endpoint returns an 11-digit phone to count as an escalated result.
2. Token exchange: use a fake code. Control: replacing the secret with an equal-length fake value should return “invalid business parameter”; the real key should change the response to “authorization code expired/invalid code.” If this site has no `/proxy`, or the token endpoint is on a separate business domain you cannot test, POST `grant_type=client_credentials` directly to the same product's independent oauth `/oauth/v2/token` (control: fake key returns `invalid_client`).
3. Distribution/open query endpoint: sign with the documented HMAC (sort keys lexicographically and concatenate; include integration parameters such as `test=test` when present). Control: fake secret fails signature; real key returns the supply list for that app.
4. Official CLI debug-package/production-package endpoint (`package/changeDebugVersion`, `changeVersion`, etc.). Control: fake key returns “key error”; a real key passing authorization and reaching “game package resource does not exist” still proves the key works; do not stop just because there is no real package uploaded. **For package-change endpoints, do not test only upload/version change**; also try WASM subpackage `packageBind` / `build` / `queryBuildResult`. Fake key should return key error; real key binding it under that application name proves write access.
5. If the gateway uses a server-IP allowlist, try `X-Forwarded-For: 127.0.0.1`. Without the header it should deny access; with it, a real key may return the supply list.
6. After a signed supply query succeeds, don't stop at the list: try pre-order validation (check), verify the real sale/settlement price, then booking. Control: fake key fails signature. Cancel an order immediately after obtaining its order number; if you cannot revert changes, stop at the response. Do not place real orders in bulk.
7. If public GitHub/SDK demo RSA+3DES is accepted by the production XML pay-gate, sign `POST /service/query` yourself and decrypt the returned ciphertext. Control: fake key fails verification. If a neighboring order reuses the same demo RSA, a business error (order number invalid) rather than verification failure still means the key is live. Do not place real orders/refund.
8. For an oversized pre-signed SDK zip in open documentation storage, extract `prod/config.properties` (and the national-crypto config) + pfx, open the certificate with the configured password, copy merchant ID/signKey, and call **production order query**. Control: fake key returns SIGN_ERROR. Query only; no real transfers.
9. Game MSDK: when wiki says encode=2, compute `sig=md5(msdkKey+timestamp)` (not bound to path/body, no time window) and sign production/test `/auth/*`. Control: fake signature returns `sig error`; real signature passing the gate and reaching a downstream business error such as identity-provider `access_token expired` / HandQ `0x711` still shows the key is live; don't call that no issue. V3 `/auth/guest_register`: plaintext UUID `reqid` returns `decrypt faild`; use the first **16 ASCII characters** of msdkKey for QQ TEA (16-round, big-endian `oi_symmetry_encrypt2`) then hex-encode. If a guestid is issued, call `/auth/guest_check_token`. Stop if there is no valid player social token; do not grind login to get into an account.
10. The official integration HTML sample `MSDKConfig.json` contains plaintext `MSDK_SDK_KEY`: when `source=0`, compute `sig=md5(path+"?"+query sorted by key excluding sig+body+key)`; decrypt with `md5(ts+ciphertext+key)` (verify the documented sample ts/ciphertext/signature match before using it in production). Control: fake signature returns `invalid sig`; after a real signature passes, try guest login; if `channel_info` contains only a self-made UUID but still issues openid/token/jwt, that is a user ticket. Do not grind channel codes to enter someone else's account.
11. Old npm package hardcodes `developerId`+`signKey`: on the partner-center gateway, fake signature for a live ID returns “signature error,” while a dead ID returns “developer data exception,” allowing live IDs to be enumerated. Real signature is `sign=SHA1(signKey+key-value pairs sorted by key)`. The privacy-number batch retrieval endpoint without a store `appAuthToken` may still return `OP_SUCCESS`; a real signature at the exchange endpoint that passes the gate and reaches “incorrect onboarding status” still proves the key works. An empty array is not a false positive (no downgraded order currently). **Do not stop**: a newer npm package from the same suite hardcodes `appAuthToken`; don't dismiss it as integration-only. Sign `queryPoiInfo` with that token, change `ePoiId`, and retrieve another merchant's store name + 11-digit phone. Do not place real orders / grind for authorization codes.
12. Recharge/enterprise-payment Demo HTML (such as `enterprise_client`) has AppId+AppKey hardcoded in an input: don't assume it is an integration placeholder. While logged out, copy them; HMAC-SHA1 source string is `GET&urlencode(path)&urlencode(key-sorted kv)`, key is `AppKey+'&'`, and Base64 gives sig. Control: fake signature returns `sig error`. With real signature call production `/v1/r/{appid}/open_order` for `token_id`. Create only unpaid orders; do not pay at checkout or refund. Actual values only in the report.
13. Login-page JS hardcodes DingTalk ISV `suiteKey`+`suiteSecret` as `clientId`/`clientSecret` (value begins `suited`): don't treat it as an OAuth placeholder. While logged out POST `https://oapi.dingtalk.com/service/get_suite_token`, with fake `suite_ticket`. Control: fake secret returns “invalid suite key or secret.” Real key returns `suite_access_token`. Actual values only in the report.
14. Do not treat a default map key in console webpack/Vuex as a placeholder. After testing a fake key as a control, don't only try reverse geocoding: call location-cloud/layer `table/list`; if a table exists, call `data/list`. Reverse geocoding alone works but table is empty → false positive. Actual values only in the report.
15. Official docs/integration HTML curl example hardcodes Gamekey: don't treat it as a placeholder. While logged out, copy AppID+Gamekey and compute `sign=md5(mod,func,appid,time,postdata,key)` to sign the production queue gateway `getZoneListCount`. Control: fake key returns `req sign error`. Real signature with `ret=0` returns region/server online status. Only query queue/online status; do not insert anyone into the queue or remove anyone from it. Actual values only in the report.

Count as success: Key is live and can be exchanged for that app's user ticket; or a business query exposes the app's supply list; or a WASM subpackage version is bound to the app; or booking creates an order number for the app; or decryption reveals an unpublished payment order and cardholder for **another merchant**; or fake signature fails verification while real signature passes production gate and reaches a downstream identity-provider business error (proving production still accepts the complete key); or production `open_order` returns an unpaid `token_id`; or DingTalk `get_suite_token` returns `suite_access_token`; or production queue `getZoneListCount` with `ret=0` returns region/server online status.

False positives: Documentation contains a placeholder; real and fake keys return the same error; key expired and cannot be tested; integration key works only in test and production rejects it; wiki masks keys and you assume zip/Demo is masked too. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Portal CMS site key hardcoded (short table has a pointer)

Recognize: The portal/open-platform frontend (not a debugging-doc zip) hardcodes Portal CMS `accessKey`+`secretKey`. There is a `GET /api/v1/cms/login` returning a JWT, commonly in `cms-token` header. Homepage may only be a CMS welcome page; key may be in another portal JS bundle.

Test (while logged out):

1. Copy AK/SK and call login (put actual values only in the report).
2. With the ticket call `channel/list`, `content/list|detail|search`, `attachment/list`.
3. Do not stop at the official site's carousel channel. Follow developer/test/admin channels and GET `attachments.url`.

Count as success: Internal test reports / article body in a channel not publicly displayed (xlsx fields count too).

False positives: Only public operation banners/download links. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Signature nonce is a private key (short table has a pointer)

Recognize: Enterprise IM/business **unlogin** signature endpoint (`queryAppSignature` / `queryCorpSignature`, etc.). Other business endpoints require login; do not use Cookie here. Response `nonceStr`/`nonce` begins with `MIIE` or `-----BEGIN` and can be loaded as a PKCS8 private key. Sometimes an unauthenticated channel/QR-code endpoint also leaks `corpId`.

Test (while logged out):

1. Copy `corpId` if leaked by an unauthenticated channel/QR-code endpoint.
2. POST the signature endpoint with url + corpId.
3. Load `nonceStr` as a private key (`load_der_private_key` in Python / openssl). Control: fake corpId should return a business error and no PEM.

Count as success: It loads as an RSA private key.

False positives: nonce is only a short random string; fake corpId always produces a business error. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Unauthenticated IM/experience ticket issuance (short table has a pointer)

Recognize: Unauthenticated `im/getConfig` issues a TIM `userSig` with `identifier=null`; or a cloud-product experience center/apaas signing endpoint issues a ticket for an **existing userId** supplied in the request (control: guest `none_auth` creates a new account); or other business endpoints in the same admin console require login while the signing endpoint needs only a custom trace header + client-supplied userId to issue IM/RTC userSig; or a custom IM gateway signing endpoint accepts only appkey and has no userId in the ticket, so the `uId` in the connect endpoint body supplies the identity; or the JS v2 inner signing endpoint is behind an IP allowlist; or official audio/video/IM Web Demo frontend hardcodes the signing URL and fixed pwd, and issues a ticket for the requested identifier even with empty password.

Test (while logged out): Use the sig to log into the `IM connection domain`: `openim/login`, `getmsg`, `friend_get`, `get_joined_group_list`, group details/messages. On experience pages also call login_token and inspect `data.phone`. Control gate: other business endpoints should require login; on signing endpoint add only the trace header and change userId to values such as interviewer_/candidate_; use authorization/signature to call group history/join room. After anonymous appkey issues a ticket, change uId in access/get to interviewer_/admin and inspect response userID. If v2 `by_app_name` returns IP not allowed, **don't stop**; try v3 same path with only `appName`. For official Demo: copy the UserSigService URL and pwd from JS, set identifier to an existing user number; try an empty pwd separately. Use userSig on friend_get and compare with a guest empty account.

Count as success: Retrieve **someone else's** session/group members/friends, or plaintext phone number.

False positives: Can log in only as guest `null`, empty session, or your own empty group; changing to someone else's UserID returns 70013; `none_auth` only creates a new empty account; demo account has no phone (doesn't count). A miss on one site does not remove this row from the short table.

### Shared JWT issued for a hardcoded product ID (short table has a pointer)

Recognize: AIGC/small-tool H5 or landing page hardcodes product ID / `qbid` in its script. Signing endpoint (`account/qb`, etc.) issues a JWT for a **shared product ID** without login; ticket name is product name, not a guest. JS may include only create / GET by ID and no list.

Test (while logged out):

1. Copy the product ID hardcoded in the page and call the signing endpoint.
2. With that ticket, POST same-origin `.../task/list` (or a list endpoint such as workflow/history absent from the JS). Control: without ticket it should require login; even empty body returns a total.
3. Follow `userImages` / original-image CDN and confirm actual faces/ID photos download.

Count as success: List has a huge total and includes **other people's** original images/ID photos.

False positives: Ticket can only create empty tasks; list total=0 or only your own uploads. Do not store actual secret values or a particular JWT. A miss on one site does not remove this row from the short table.

### Official customer-service chain signs for an external site (short table has a pointer)

Recognize: Enterprise customer-service H5 signing chain; inline code has `getXcxLink`/`get_wx_open_link`. This is not a generic redirect page.

Test (while logged out): POST `queryStr=caUrl=external-site&urlType=2`. Open the returned `customer-service landing chain`.

Count as success: Landing page's formal name is the official customer-service account, and `base64Decode` of the query contains the URL you supplied.

False positives: Signs only for this site's `/ca/`; landing page has no query; login is required. A miss on one site does not remove this row from the short table.

### Demo account obtains cloud credentials (short table has a pointer)

Recognize: Cloud vendor product Demo / experience page / console proxy / supplier backend; STS, federated identity, or task creation requires only business ID or empty body, not Cookie. Missing parameters on STS endpoint return field validation, not necessarily a login gate; a demo account may not exist. This is separate from object-storage STS wildcard overwrite (see `file-upload-test.md`).

Test (while logged out): If a demo account exists, copy the customer ID and call credential issuance / task creation / federated identity. If no demo account exists, still test the credential endpoint: don't mistake missing-field validation for a login gate. If empty body says `AppId`/`Uin` missing, that still may be an endpoint; supply a parseable numeric Uin and it may issue federated STS; continue with bucket+file_name. Credential endpoint may be under a separate demo CDN `/openapi/` (follow the page JS). After the share domain frontend HMAC passes the gateway, compare with other endpoints that say “invalid token / Token validation failed”; still test an STS endpoint such as `/share/token/info` without the share-page token. Change bucket and use PUT/NoSuchBucket to probe buckets. Use the credentials with GetCallerIdentity to identify the main account; if actions such as launch/search exist, sign them too (if quota is low, don't stop; check whether you can learn the account or retrieve billing data). After Epaas HMAC `SC-SIGN` passes, compare with list endpoints that still return UserNotLogin; specifically look for GetUploadURL/FileName that do not validate the token (passing HMAC is not data access).

Count as success: Temporary credentials reveal AccountId/role name; list under the same identity shows a task just created; search endpoint returns billing/instance/other-entity business fields; or live bucket accepts PUT 200.

False positives: Credentials get Unauthorized on business APIs; credential endpoint requires login/401; create returns 200 but nothing appears in list. No demo account/customer ID is not a false positive. A miss on one site does not remove this row from the short table. Do not store actual secret values.

### Unauthenticated partner-login chain issuance (short table has a pointer)

Recognize: Electronic-contract / supplier portal opens on a login page; field accepts only an enumerable numeric partner ID (partnerId / supplierId, etc.), with no password. Unauthenticated `getUrl` (or similar login-chain issuer) directly returns `partnerId`+`generate`+`code`. Nearby `setCookies` / token exchange consumes these three values and issues a session.

Test (while logged out):

1. Copy issuer endpoint from login-page JS; use the numeric partner ID shown on page or a neighboring ID.
2. POST the three returned fields to setCookies / token exchange; copy Set-Cookie.
3. With the session, call contract list / payment onboarding / identity endpoint. Control: without Cookie it should be unauthenticated; nonexistent partner ID should return no result.

Count as success: Identity endpoint logs in successfully, or you can read unpublished contracts as that account.

False positives: Issuance endpoint requires login; code must be obtained by clicking an email; setCookies issues no session; only public partner recruitment page. A miss on one site does not remove this row from the short table. Do not store actual secret values or a particular Cookie.

### Unauthenticated direct query of partner configuration (short table has a pointer)

Recognize: Operations / supply-chain frontend exposes a direct supplier-configuration query without login. Body contains only an enumerable numeric partner ID (`partnerId`, etc.); no Cookie. Response `partnerInfos` (or equivalent) includes clientId+clientSecret.

Test (while logged out):

1. Copy the query endpoint from operations-console JS (getV2 / tech/partner, etc.).
2. Try a neighboring partnerId. Control: an unconfigured ID should return empty/not configured, not a login gate.
3. Confirm the result is a complete clientSecret, not an empty success code.

Count as success: Response contains complete clientId+clientSecret usable as direct credentials.

False positives: Endpoint has been removed; clientSecret is empty or a placeholder; login required; only company name and no key. A miss on one site does not remove this row from the short table. Do not store actual secret values.

### Unauthenticated issuer redirects to an external domain (short table has a pointer)

Recognize: Unauthenticated SSO/redirect issuer; callback only checks whether the string contains the official host, or is not validated and any external domain can be signed.

Test (while logged out): Set callback/redirect to an external domain; try both with the official host embedded (query/subdomain/userinfo) and **without it**. If the response includes a ticket/signature, pass it to your own SSO. Check whether the pass-through page does `location.replace(redirect+"?"+tokenName+"="+session)`. Control: a value without the official string should be cleared or blocked. If a real device login is required to issue a ticket and frontend does not append the token to the redirect → partial; stop at the signed link, do not count it as full compromise.

Count as success: SSO succeeds while callback remains external, or after login the pass-through page sends the session to the external domain's query.

False positives: Callback gets cleared; SSO lacks callback parameter. A miss on one site does not remove this row from the short table.

### Missing field supplied in header/Cookie (short table has a pointer)

Recognize: An unauthenticated business endpoint returns JSON “missing xxx”; putting it in query/body still says missing. Or Spring 400 HTML/`message` names `Required String parameter 'os' is not present`, where the named item is a **Cookie name**, not a query parameter.

Test: Put this field in an **HTTP header** and try SSO/token exchange/login. If the header is ignored, don't stop; try it as a Cookie (make up a device `uuid`/`os`/`platform`, etc.). Also try unauthenticated write endpoints (bind store/change status), not just token exchange. Control: query-only should still report missing.

Count as success: Get another person's login ticket/name, or write another person's store/object.

False positives: Query already has the field but it is still missing (header and Cookie both ignored) does not count; only your own ticket returned. A miss on one site does not remove this row from the short table.

### Username/password endpoint behind an SSO shell (short table has a pointer)

Recognize: Admin backend opens only by redirecting to SSO/unified auth, with no username/password form; backend still exposes a username/password login API. Do not treat the SSO shell as proof there is no login endpoint. A common backend skin is `/admin-api/system/auth/login`; test even if this name is absent.

Test (while logged out):

1. POST `/admin-api/system/auth/login`, body `{"username":"admin","password":"123456"}` (do not grind the SSO CAPTCHA).
2. Put `data.accessToken` in `Authorization: Bearer` and call `/admin-api/system/tenant/page` or `/system/user/page`.
3. Control: wrong password should fail; a default endpoint returning a platform ticket with `userId=1` is a clue.

Count as success: Access platform-admin account and tenant/user list exposes other people's phone/name.

False positives: Login endpoint 404 or default account has been changed; ticket only reaches an empty tenant with no contacts. Do not store actual secret values or a particular token. A miss on one site does not remove this row from the short table.

### Empty refresh request issues an administrator ticket (short table has a pointer)

Recognize: JSON gateway has a refresh-ticket endpoint (such as `/api/auth/refresh`). While logged out, an empty body `{}` issues a ticket whose identity is `admin` (not guest). Admin backend and frontend use the same cloud development/gateway stack.

Test (while logged out): POST refresh endpoint; put `data.token` in `Authorization: Bearer` and call `/api/admin/me/permissions` and application/user lists. Control: without this ticket they should require login.

Count as success: `me` is a super admin (permissions `*`) or list exposes **other people's** phone/name.

False positives: Only guest/expired ticket; refresh requires an old refresh token; `me` is still your own account. Do not store actual secret values. A miss on one site does not remove this row from the short table.

### Empty openId enters an existing account (short table has a pointer)

Recognize: Mini-program/H5 GET login endpoint accepts `openId` (`loginByOpenId`, etc.). Missing parameter reports the missing field, fake openId fails login, but empty string still returns 200 with an **existing merchant account** and phone number, not an empty guest account.

Test (while logged out):

1. Try once with parameter missing to confirm it is a login endpoint.
2. Try a fake value; it should fail.
3. Try `openId=` as an empty string; if GET query is ignored, try POST JSON `{"openId":""}`.
4. Compare: store name/phone returned for empty string should be an existing store/login phone, different from fake-value failure and missing-parameter response, not the same guest account.

Count as success: Enter existing merchant account and retrieve its phone (can act as that account).

False positives: Empty string returns only a guest empty account. Do not store actual secret values or a particular Cookie. A miss on one site does not remove this row from the short table.


