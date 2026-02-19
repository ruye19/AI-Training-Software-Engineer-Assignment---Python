# Explanation

## What was the bug?

For API requests, the client should refresh the OAuth2 token when the token is missing, invalid, or expired, and then set the `Authorization` header using the (possibly new) token. When `oauth2_token` was a **plain dict** (e.g. from an old API shape), the code did **not** refresh. It only refreshed when `oauth2_token` was falsy or when it was an `OAuth2Token` instance that was expired. So a truthy non-`OAuth2Token` value (like a dict) was never refreshed and never used for the header, and the request went out without `Authorization`.

## Why did it happen?

The condition was:

- `if not self.oauth2_token or (isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired):`

For a dict, `not self.oauth2_token` is false and `isinstance(..., OAuth2Token)` is false, so the whole condition was false and `refresh_oauth2()` was never called. The header was only set when `isinstance(self.oauth2_token, OAuth2Token)` was true, so the dict was ignored and no header was set.

## Why does your fix solve it?

The condition was changed to:

- `if not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired:`

So we refresh whenever the token is **not** a valid `OAuth2Token` (including `None` or a dict) or when it **is** an `OAuth2Token` but expired. After refresh, `oauth2_token` is always an `OAuth2Token`, so the header is set correctly. Minimal change, no refactor.

## One realistic case your tests still don't cover

**Clock skew or boundary:** The tests do not cover the exact moment when `expires_at` equals the current time (e.g. token expires at T and the check runs at T). Depending on how “expired” is defined (e.g. `>=` vs `>`), this could be off-by-one in rare cases. A test that mocks or freezes time at that boundary would make that behavior explicit.
