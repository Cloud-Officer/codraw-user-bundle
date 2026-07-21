# Code Review — codraw/user-bundle

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
**composer.json (M7, H1)**
- Moved `codraw/security` from `require-dev` to `require` (`^0.39`) — `Draw\Component\Security\Core\Security` is used by always-on services (`Feed/FlashUserFeed.php`, `EventListener/UserRequestInterceptorListener.php`).
- Added `symfony/password-hasher` `^6.4.0` to `require` — hard-imported by the default-enabled `EventListener/EncryptPasswordUserEntityListener.php` and `Controller/Api/ConnectionTokensController.php`.
- Added `suggest` entries for the optional/feature-gated integrations flagged in M7 and H1: `codraw/mailer`, `codraw/open-api`, `codraw/sonata-integration-bundle`, `symfony/console`, `symfony/expression-language`, `symfony/mailer`, `symfony/messenger`, `symfony/security-bundle`, `symfony/twig-bridge`, `symfony/validator`. These are only reached when a disabled-by-default feature is turned on (account_locker, onboarding, enforce_2fa, password_change_enforcer, email_writers), when the API controller routes are imported, or via guarded code paths, so they stay suggestions rather than hard requirements.
- `composer validate --no-check-publish` passes. The `php` constraint was already present as `>=8.5`.

### Code fixes

- **H1** — `DrawUserBundle.php`: guarded the `AdminLoginFactory` registration with `class_exists(AdminLoginFactory::class)`; the container no longer fatals at build time when `codraw/sonata-integration-bundle` is not installed (also added the package to `suggest`).
- **M1** — `EventListener/EncryptPasswordUserEntityListener.php`: auto-generated passwords now use `bin2hex(random_bytes(16))` instead of the predictable `uniqid()`. The plain value is only hashed and then discarded (`postPersist`/`postUpdate` clear it), so the format change is invisible to consumers.
- **M3** — `Security/TwoFactorAuthentication/Entity/ConfigurationTrait.php`: `enableTwoFActorAuthenticationProvider()` now calls `\in_array($provider, $enabledProviders, true)` — enabling an already-enabled provider no longer re-runs the setter (which wiped a pending TOTP secret).
- **M4** — `Feed/FlashUserFeed.php`: nullable translator is now null-safe (`$this->translator?->trans(...) ?? $message`); falls back to the raw message instead of a call-on-null error.
- **M5** — `MessageHandler/NewUserSendEmailMessageHandler.php`: added a `null === $user` guard so a deleted user skips the message instead of a `TypeError` in `method_exists(null, ...)`.
- **M6** — `EmailWriter/PasswordChangeRequestedEmailWriter.php`: now resolves the user via the entity repository (`find($email->getUserId())`), consistent with the other writers, instead of passing a database id to `UserProviderInterface::loadUserByIdentifier()` (which expects the user identifier / e-mail). *Open item:* the hardcoded `admin_change_password` route is unchanged — wiring it to `password_change_enforcer.change_password_route` is a config/design decision.
- **L1** — `EventListener/UserRequestInterceptorListener.php`: `getFirewallConfig()` result is now null-safe (`$firewallConfig?->isStateless()`); requests outside any firewall no longer fatal.

### Validation pass (2026-07-20)

- `composer install` (CI flags) resolves cleanly with the updated `composer.json` — no constraint adjustment needed.
- Full PHPUnit run against MySQL (`codraw_user_bundle` DB): 163 tests, 239 assertions, 0 failures/errors. The 11 PHPUnit notices ("no expectations configured for mock") in `Tests/EventListener/TwoFactorAuthenticationListenerTest.php` are pre-existing (identical count with the fixes stashed) and do not affect the exit code.
- PHPStan (level 5): 41 errors, all pre-existing on unmodified master (42 before this pass — the H1 `class_exists` guard resolved the `DrawUserBundle.php:20` "Instantiated class AdminLoginFactory not found" error; the remaining `argument.type` error on that line is pre-existing and stems from `codraw/sonata-integration-bundle` not being installed, so PHPStan cannot see the class). The baseline (`phpstan-baseline.neon`) is empty; no stale entries to prune. No new errors were introduced by the fixes.
- markdownlint-cli2: fixed two MD036 violations in this file (bold "Code fixes" / "Well covered" pseudo-headings converted to real `###` headings); all 6 markdown files now lint clean. No tracked markdown files needed changes.
- No test-expectation updates were required: no existing test pinned the old buggy behaviors changed by H1/M1/M3/M4/M5/M6/L1.

**Not fixed (deliberately, too risky for a scoped pass)**: H2 (preUpdate change-set semantics — requires reworking the listener signature and functional verification of the persistence flow), M2 (authentication response behavior change), M8 (changing the config default would break consumers relying on `App\Entity\User`), L2 (narrowing the caught exception changes runtime behavior), L3/L4/L5 (design decisions / public-API BC breaks).

## Overall Assessment

This bundle provides user-security features (account locking, password change enforcement, onboarding e-mails, 2FA enforcement, JWT connection tokens) for the Draw/Codraw framework. The code is modern (PHP 8 attributes, constructor promotion, strict signatures), the DI extension cleanly toggles whole feature sets on/off, and there is a good, deliberate design around decoupling the user entity via `resolve_target_entities` and message-based side effects. However, the review found two high-severity problems — an undeclared hard dependency on `codraw/sonata-integration-bundle` inside the bundle's `build()` method, and a Doctrine `preUpdate` password-hashing listener that mutates the entity without updating the change set — plus a cluster of medium issues: a predictable `uniqid()` auto-generated password, user enumeration in the connection-token endpoint, a wrong-arguments `in_array()` call in the 2FA configuration trait, a null-translator dereference, and several undeclared runtime dependencies. Test coverage is decent for DI configuration and lock entities, but the most security-sensitive classes (password listener, controller, account locker service) are untested.

Overall grade: **C** — good architecture with several real, verified bugs in seldom-exercised but important paths.

---

## Findings

### High

#### **[FIXED]** H1. `DrawUserBundle::build()` hard-depends on a class from an undeclared package

`DrawUserBundle.php:5,17-21`

```php
use Draw\Bundle\SonataIntegrationBundle\DependencyInjection\Factory\Security\AdminLoginFactory;
...
if ($container->hasExtension('security')) {
    $extension = $container->getExtension('security');
    $extension->addAuthenticatorFactory(new AdminLoginFactory());
}
```

`AdminLoginFactory` lives in `codraw/sonata-integration-bundle` (verified: `codraw-sonata-integration-bundle/DependencyInjection/Factory/Security/AdminLoginFactory.php`), which is **not** listed in `require`, `require-dev`, or even `suggest` of this package's `composer.json`. Any application that installs this bundle together with the Symfony security bundle (i.e., virtually every consumer of a *user* bundle) but without the Sonata integration bundle will get a fatal "class not found" error at container build time. There is no `class_exists()` guard. Either declare the dependency, guard with `class_exists(AdminLoginFactory::class)`, or move this wiring into the Sonata integration bundle where it belongs.

#### H2. Password hash set in Doctrine `preUpdate` is not part of the change set and may never be persisted

`EventListener/EncryptPasswordUserEntityListener.php:18-21,38-54`

The listener is tagged for `preUpdate` (`DependencyInjection/DrawUserExtension.php:135`) and does:

```php
public function preUpdate(SecurityUserInterface $user): void
{
    $this->updatePassword($user); // ... $user->setPassword($hash)
}
```

Per Doctrine ORM semantics, changes made to mapped fields inside `preUpdate` are **ignored by the flush in progress** — the persister writes only the already-computed change set; you must use `PreUpdateEventArgs::setNewValue()` (or recompute the change set) for the new `password` value to be included in the UPDATE statement. As written, when an existing user's `plainPassword` is set, the flush is triggered only because `passwordUpdatedAt` changed (`Entity/SecurityUserTrait.php:86-97`), and the freshly hashed `password` is *not* written in that flush. It only reaches the database if a second flush happens later in the request. Password changes for existing users can therefore be silently lost depending on the application flow. The `prePersist` path (new users) is fine — `prePersist` changes are persisted. Fix: accept `PreUpdateEventArgs` and call `$eventArgs->setNewValue('password', $hash)` (and `hasChangedField` guard), or hash in a `onFlush`/`prePersist`-plus-recompute approach.

### Medium

#### **[FIXED]** M1. Auto-generated passwords use `uniqid()` — predictable, low entropy

`EventListener/EncryptPasswordUserEntityListener.php:47`

```php
$user->setPlainPassword(uniqid());
```

`uniqid()` is derived from the current microtime; it is documented as *not* cryptographically secure. An attacker who knows the approximate account-creation time can enumerate the small candidate space offline (or online, absent throttling). Since this password is set on any user persisted without a password when `auto_generate_password` is enabled (the default), use `bin2hex(random_bytes(16))` or `ByteString::fromRandom()` instead.

#### M2. User enumeration and no throttling on the connection-token endpoint

`Controller/Api/ConnectionTokensController.php:45-56`

`createAction()` returns HTTP 400 "User not found" when the identifier does not exist and HTTP 403 "Invalid credential" when the password is wrong, which lets an attacker enumerate valid usernames. The response-time difference (no password hashing performed on the unknown-user path) is an additional oracle. The endpoint also performs raw password verification with no integrated rate limiting/login throttling (it bypasses the firewall's `login_throttling` since it's a plain controller). Return a single 401/403 "Invalid credentials" for both cases, run a dummy hash verification on the unknown-user path, and document/enforce a rate limiter.

#### **[FIXED]** M3. Wrong arguments to `in_array()` in `enableTwoFActorAuthenticationProvider()`

`Security/TwoFactorAuthentication/Entity/ConfigurationTrait.php:32-41`

```php
$enabledProviders = $this->getTwoFactorAuthenticationEnabledProviders();

if (!\in_array($enabledProviders, $this->twoFactorAuthenticationEnabledProviders, true)) {
    $enabledProviders[] = $provider;
    $this->setTwoFactorAuthenticationEnabledProviders($enabledProviders);
}
```

The needle is the *array of providers* instead of `$provider`, so the condition is effectively always true and the setter always runs. Duplicates are masked by `array_unique()` in the setter, but the buggy path has a real side effect: `setTwoFactorAuthenticationEnabledProviders()` (line 27-29) wipes `totpSecret` whenever `totp` is not in the list — so calling `enableTwoFActorAuthenticationProvider('email')` while a TOTP enrollment is in progress (secret stored, `totp` not yet confirmed) silently destroys the pending TOTP secret. Should be `\in_array($provider, $enabledProviders, true)`.

#### **[FIXED]** M4. `FlashUserFeed` dereferences a nullable translator

`Feed/FlashUserFeed.php:17,34`

The constructor accepts `private ?TranslatorInterface $translator`, but `addToFeed()` calls `$this->translator->trans(...)` unconditionally. In an application without a translator service the autowired argument is null and every feed addition throws a `TypeError`/`Error` (call on null). Either require the translator or fall back to the raw message when it is null.

#### **[FIXED]** M5. `NewUserSendEmailMessageHandler` crashes when the user no longer exists

`MessageHandler/NewUserSendEmailMessageHandler.php:27-29`

```php
$user = $this->drawUserEntityRepository->find($message->getUserId());
if (!method_exists($user, 'getEmail') || empty($user->getEmail())) {
```

`find()` returns `null` when the user was deleted before the (asynchronous) message is handled; `method_exists(null, 'getEmail')` throws a `TypeError` on PHP 8, so the message fails and is retried/dead-lettered instead of being skipped. Compare with `PasswordChangeRequestedSendEmailMessageHandler.php:30`, which handles null correctly via an `instanceof` check. Add a `null === $user` guard.

#### **[FIXED]** M6. `PasswordChangeRequestedEmailWriter` passes a user *id* to `loadUserByIdentifier()`

`EmailWriter/PasswordChangeRequestedEmailWriter.php:32`

```php
$user = $this->userProvider->loadUserByIdentifier($email->getUserId());
```

`PasswordChangeRequestedEmail::getUserId()` is populated with `$user->getId()` (`MessageHandler/PasswordChangeRequestedSendEmailMessageHandler.php:40`; `Message/PasswordChangeRequestedMessage.php` `preSend()`), while `SecurityUserTrait::getUserIdentifier()` returns the e-mail. Unless the application's user provider happens to resolve database ids, this throws `UserNotFoundException` inside e-mail composition and the "password change requested" e-mail fails to send. The other writers (`ForgotPasswordEmailWriter`, `ToUserEmailWriter`) consistently use the entity repository; this one should too. It also hardcodes the `admin_change_password` route (line 43) instead of using the configured `password_change_enforcer.change_password_route`.

#### **[FIXED]** M7. Several runtime dependencies are not declared in `composer.json`

`composer.json:11-45`

Classes that are registered as services by default reference packages that are only in `require-dev` or absent entirely:

- `Controller/Api/ConnectionTokensController.php` — `Draw\Component\OpenApi\*` (`codraw/open-api`, not declared anywhere) and `Draw\Component\Security\Http\Authenticator\JwtAuthenticator` (`codraw/security`, dev-only), `symfony/expression-language`.
- `Feed/FlashUserFeed.php`, `EventListener/*` — `Draw\Component\Security\Core\Security` (`codraw/security`, dev-only).
- `Command/RefreshUserLocksCommand.php` — `symfony/console`; message handlers — `symfony/messenger`, `symfony/mailer` (`MailerInterface`); `Email/TwoFactorAuthCodeEmail.php` — `symfony/twig-bridge`; `Entity/SecurityUserTrait.php`, `DTO/Credential.php` — `symfony/validator`.
- `LockableUserTrait` / `OnBoardingLifeCycleHookUserTrait` use `Draw\Component\Messenger\...\MessageHolderTrait` and `function Draw\Component\Core\use_trait` (`codraw/messenger` is dev-only/suggested; `codraw/core` is declared — good).

Most of these "work" today because a typical Draw application installs everything, but the package is not installable standalone with its declared dependency set. Promote the truly required ones (at minimum `codraw/security` given `Security` is used by always-on listeners, and `symfony/console` / `symfony/messenger` when account-locker features are enabled) or guard registration with `class_exists`/`interface_exists` as done for `EmailWriterInterface` (`DependencyInjection/DrawUserExtension.php:178`).

#### M8. Bundle configuration defaults to `App\Entity\User`

`DependencyInjection/Configuration.php:5,45-51`

A reusable bundle imports `App\Entity\User` and uses it as the default for `user_entity_class`. In any application without that exact class, the default value fails the `class_exists` validation with a confusing "The class [App\Entity\User] for the user entity must exists." error even though the user never set it, and it couples the bundle to the app skeleton namespace. Prefer no default plus `->isRequired()`, which produces a clearer error.

### Low

#### **[FIXED]** L1. Possible null dereference on `FirewallMap::getFirewallConfig()`

`EventListener/UserRequestInterceptorListener.php:120-124`

`getFirewallConfig()` is `?FirewallConfig`; for a request not matched by any firewall the subsequent `$firewallConfig->isStateless()` would fatal. Reachable only in unusual setups (authenticated user token present outside a firewall context), hence low.

#### L2. `EmailTwoFactorProvider` swallows every `Throwable`

`Security/TwoFactorAuthentication/Email/EmailTwoFactorProvider.php:26-33`

`validateAuthenticationCode()` catches `\Throwable` and returns `false`. The intent is to convert `ByEmailTrait::getEmailAuthCode()`'s `LogicException` (`Security/TwoFactorAuthentication/Entity/ByEmailTrait.php:31-38`) into a failed validation, but this also hides real infrastructure errors (DB failures, typos) as "wrong code". Catch the specific exception type. Related: `getEmailAuthCode(): string` throws where scheb's `TwoFactorInterface` contract is nullable — the decorator is compensating for an LSP violation in the trait.

#### L3. `DELETE /connection-tokens/current` is a no-op

`Controller/Api/ConnectionTokensController.php:75-81`

`clearAction()` has an empty body: the JWT is not invalidated server-side (stateless tokens can't be, without a denylist). Exposing a DELETE endpoint that silently does nothing gives API consumers a false sense of revocation. Either implement a token denylist or document clearly that the token remains valid until `exp`.

#### L4. Side-effectful getters used as Doctrine lifecycle callbacks

`Entity/UserLock.php:48-56,97-101`

`getId()`/`getCreatedAt()` are annotated `#[ORM\PrePersist]` and lazily initialize state. It works, but getters that mutate and double as lifecycle hooks are surprising; a dedicated `#[ORM\PrePersist] public function onPrePersist(): void` would be clearer. Similarly `UserLock::isActive()` (`Entity/UserLock.php:154-165`) relies on an unusual `switch (true)` with a leading `default:` fall-through that most readers will misparse.

#### L5. Public API typos

`Security/TwoFactorAuthentication/Entity/ConfigurationTrait.php:32,43,48` (`enableTwoFActorAuthenticationProvider`, `disableTwoFActorAuthenticationProvider`, `asOneTwoFActorAuthenticationProviderEnabled` — presumably "hasOne..."), `EventListener/PasswordChangeEnforcerListener.php:26` (`checkNeedNeedChangePassword`), `AccountLockerListener.php:65,72` (`handlerGetUserLocksEvent`, `handlerCheckPreAuthEvent`), and the (external) `JwtAuthenticator::generaToken()` call in `ConnectionTokensController.php:58,69`. These are public/protected API names that are painful to fix later without BC breaks.

---

## Strengths

- **Clean feature toggling in the DI layer**: each feature (`account_locker`, `enforce_2fa`, `password_change_enforcer`, `email_writers`, `onboarding`) removes all of its service definitions when disabled, and even excludes the `UserLock` entity from the Doctrine metadata driver via a compiler pass (`DependencyInjection/DrawUserExtension.php:221-241`, `DependencyInjection/Compiler/ExcludeDoctrineEntitiesCompilerPass.php`) — disabled features leave no schema or service residue.
- **User-entity decoupling** through `resolve_target_entities` prepending (`DrawUserExtension::prepend()`), interfaces + traits (`SecurityUserTrait`, `LockableUserTrait`), letting applications bring their own user class.
- **Anti-enumeration design in the forgot-password flow**: when the e-mail is unknown, a different e-mail ("user not found" template with an invite link) is still sent to the requested address instead of leaking account existence (`EmailWriter/ForgotPasswordEmailWriter.php:42-61`).
- **Thoughtful lock lifecycle**: delayed lock activation via `DelayStamp` computed from `lockOn`, `DispatchAfterCurrentBusStamp` for transactional safety, and searchable stamps for message replacement (`MessageHandler/UserLockLifeCycleMessageHandler.php`); `temporaryUnlockAll` preserves `unlockUntil` across lock refreshes (`Entity/LockableUserTrait.php:62-65`).
- **Modern, consistent code**: PHP 8 attributes everywhere, constructor property promotion, readonly-style DTOs, an *empty* PHPStan baseline (`phpstan-baseline.neon`), and interception events (`UserRequestInterceptionEvent`) with a clear reason/response protocol that composes well across listeners with explicit priorities.

---

## Test Coverage

Roughly 1,250 lines of tests for ~2,500 lines of source; coverage is uneven:

### Well covered

- DI extension and configuration: nine test classes covering default service registration and each feature flag permutation (`Tests/DependencyInjection/*`), plus `ConfigurationTest` for the config tree.
- Lock domain model: `LockableUserTraitTest` (274 lines) and `UserLockTest` with a data provider covering all `isActive()` states — the tricky `switch(true)` logic is pinned down.
- `TwoFactorAuthenticationListenerTest` (329 lines) covers the request-interception 2FA flow.
- `TemporaryUnlockedMessageTest`, and a (thin) `ConfigurationTraitTest` for provider enable/dedup.

**Not covered** (notably where the real bugs are)

- `EncryptPasswordUserEntityListener` — no test at all; the H2 preUpdate persistence bug and the M1 `uniqid()` password would both be surfaced by a functional test with a real EntityManager.
- `AccountLocker` service, `AccountLockerListener`, `UserRequestInterceptorListener`/`UserRequestInterceptedListener` (session redirect round-trip), `PasswordChangeEnforcerListener`.
- `ConnectionTokensController` (authentication logic, error codes), all four `EmailWriter` classes (M6 would be caught), all `MessageHandler` classes (M5 would be caught), `FlashUserFeed` (M4), `RefreshUserLocksCommand`, `AuthCodeMailer`, `QrCodeGenerator`, `EmailTwoFactorProvider`.
- `ConfigurationTrait::enableTwoFActorAuthenticationProvider()` — the existing trait test exercises only the setter, not the buggy enable path (M3).

Recommendation: prioritize functional tests around the password lifecycle (persist + update through a real `EntityManager`) and the connection-token endpoint, since those are the security-critical paths currently untested.
