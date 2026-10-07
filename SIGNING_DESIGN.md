# Signed cloud fan controls — experimental design for review

This contribution implements opt-in signed/encrypted fan commands on cloud-paired
X1C firmware. **It is open for normal review, not a draft.** It is not yet a turnkey HACS
release; remaining trust, licensing and lifecycle limitations are documented
below rather than used to defer implementation.

## Evidence and credit

The deployed development build used ha-bambulab 2.2.25, Home Assistant 2026.9.1
on ARM64, and an X1C on firmware 01.12.00.00 with cloud MQTT and Developer LAN
Mode off. An idle-printer chamber-fan test observed 0 → 20 → 0% in both the
fan entity and the separate printer speed sensor. Restoration was confirmed.
The part-cooling and auxiliary entities were available but were not separately
actuated in that controlled test. The owner subsequently confirmed the controls
worked in normal use. No claim is made for other printer models or firmware.

This review branch ports that implementation onto current `main`; it is not a
snapshot of an installation. Its tests use generated fixtures and mocked
clients. Software contract coverage is distinct from the historical hardware proof.

Protocol credit belongs to [Open Bamboo Networking](https://github.com/ClusterM/open-bamboo-networking),
particularly research/06.02-mqtt.md and research/10.02–10.04. Local credential
acquisition used [BambuSlicerKeySaver](https://github.com/danielwoz/BambuSlicerKeySaver).
No claim is made to have discovered Bambu's signing protocol.

## Behavior

- Native part-cooling, auxiliary and chamber fan controls and ordinary HA fan
  actions; no heatbreak control or additional motion/heater entities.
- Missing or invalid optional signing material leaves telemetry working.
- `security.app_cert_install` provisioning correlates responses; device trust
  is invalidated on disconnect and re-established for the next session.
- G-code uses encrypted `param_enc` only, never plaintext `param` alongside it.
  The exact serialized payload and byte length are signed with RSA/SHA-256.
- Command IDs are reserved durably under a process lock before publication;
  they do not reset at reconnect/restart or wrap at 30000.
- Unavailable authorization and failed publication propagate errors to HA.
  Authorization rejection triggers reprovisioning, **not command replay**.
- Signed fan state comes from printer telemetry, not an optimistic override.
  Slicer G-code may overwrite a requested speed during printing.

## Credential and operational boundary

The prototype reads operator-provided `slicer_key.pem`, `slicer_cert.pem` and
`slicer_crl.pem` from `/config/.storage/bambu_lab_signing/PRINTER_SERIAL/`.
It requires a private directory and regular owner-only, non-symlink files.
Preserve `sequence.json` and its lock across restarts and credential rotation.
`python tools/check_signer.py /path/to/private/bundle` validates offline and
returns only status, not key contents. No account token, device identity,
credential, proprietary binary, capture, or deployment receipt is supplied.

**Provisioning is not renewal.** Session provisioning is implemented, but
vendor credential refresh, onboarding/repair UX and renewal are unresolved.
Without acceptable material, controls remain unavailable.

### Stale vendor CRL: explicit limitation

Two official Linux plugin versions supplied the same signed CRL beyond its
`nextUpdate`. That is not evidence of fresh revocation status. Strict expiry
validation is the default. The hardware proof used a private, explicitly
reviewed compatibility receipt, pinning the exact certificate and CRL hashes
for at most 30 days. Signature, certificate validity, key matching and known
revocation checks still apply, but later revocations cannot be ruled out.
No operator's receipt or hashes are included. This policy is included for
honest review, **not proposed as an automatically renewed default**.

## Licensing and maintainer decisions

The signing module, signing fixtures and standalone validator are supplied
under AGPL-3.0-only (`COPYING.signing`). Existing upstream MIT notices remain
intact; this contribution does not silently claim those additions are MIT or relicense
the upstream project. Distribution of a combined work needs the corresponding
AGPL obligations considered before merge.

The preferred topic for maintainer feedback is an independently packaged AGPL
signer companion with a narrow transport boundary, versus a separately
maintained AGPL fork. A process boundary alone does not settle licensing.
An in-process merge should wait for an explicitly acceptable licensing basis.

Other decisions before readiness:

1. Credential acquisition, rotation, revocation and repair ownership.
2. Trust validation for device certificates and provisioning replies.
3. Explicit user opt-in and supported-model/capability gating.
4. Whether any bounded stale-CRL policy is acceptable at all.
5. Reboot/reconnect/rejection behavior and broader hardware coverage.

### Concrete implementation gaps (7 September review)

- Printer replies are correlated to the pending provisioning sequence, and the
  returned certificate is checked for dates and an RSA key. This is **not**
  certificate-chain validation or binding that certificate to the intended
  printer identity. Define the trust anchor/identity policy and add negative
  tests before treating provisioning as authenticated device trust.
- Placing a bundle at the expected private path currently acts as opt-in.
  There is no config/options flow for consent, supported-model selection,
  credential import, renewal, or repair. File placement is a prototype setup
  mechanism, not the proposed finished user experience.
- **Resolved in the 22 September revision:** the signer accepts only a fan ID
  (part cooling, auxiliary, chamber) and a finite 0–100 percentage. It builds
  the sole canonical M106 command internally. Generic `print` publication on
  secured firmware is rejected without signing or publication; read-only and
  light traffic, and unsigned-printer behavior, remain unchanged. Secondary
  auxiliary and heatbreak fans are outside the signed capability set.
- A stale-CRL receipt cannot prove current revocation status. Do not mark this
  ready by simply rolling its deadline forward; renewal needs a supported
  source and an explicit policy for when that source is unavailable.

The independent fan correctness changes in #2120 have merged. This branch is
synchronized with `main` after that merge; those fixes no longer form part of
its feature diff. Synchronizing the branch does not resolve the gates above.

## Tests

`python -m pytest tests/pybambu -q` exercises the existing regression suite and
generated signing fixtures (encryption, signatures, CRLs, permissions,
correlation, sequence persistence/concurrency, rejection and no replay).
`python -m pytest tests/test_fan_entity.py -q` runs additional mocked entity
checks when Home Assistant is installed; otherwise those checks skip.
No test needs vendor credentials, makes a service call or controls a printer.

## Fan-only review contract (22 September)

`BambuClient.publish_fan(FansEnum, percentage)` is the only signed transport
entry. `CommandSigner.sign_fan_command(fan_id, percentage)` constructs the
payload internally and reserves a durable sequence only after input validation.
No caller-supplied payload, G-code, encrypted field or extra command can enter
that interface. Percentages retain the integration's ten-percent rounding.
A broker error returns false; no automatic command retry is introduced.
This is a scope boundary, not a sandbox against hostile code in the HA process.

`tests/pybambu/test_fan_signing_boundary.py` is a credential-free contract suite
using newly generated keys/certificates. It decrypts every supported fan's
output at representative speeds and tests invalid/nonfinite/boolean values,
unsupported fan IDs, compound commands, generic publication refusal, unsigned
compatibility, broker failure and no replay. The existing signing suite covers
reconnects, sequence persistence, expiry and rejection. These synthetic
self-signed peers do **not** establish real-printer certificate trust.

Still blocked: independent trust anchors and device identity policy, complete
application-chain validation, explicit setup/repair UX, a fresh revocation
source, renewal, and acceptable licensing/architecture. No stale-CRL deadline
was extended and no licensing notice was removed in this revision.

## Explicit consent and lifecycle diagnostics (7 October)

The options flow now exposes **Enable experimental signed fan controls** in
Advanced options for both cloud and LAN entries. It is disabled by default,
including upgrades of existing entries. Only the literal boolean `True` enables
the credential path; directory/file presence, strings and numeric values do
not count as consent. Disabling it reloads the entry and prevents further
signed commands, while ordinary telemetry/light traffic remains unchanged.

The operator still supplies the private bundle at the documented fixed path.
Configuration forms never accept or display PEMs, keys or arbitrary paths.
Diagnostics export only allowlisted readiness, review deadline and stable
reason codes: disabled, credentials_missing, credentials_invalid,
credentials_expired, provisioning_required, ready. The private bundle path is
redacted. A deadline within seven days is flagged; diagnostic generation does
not renew credentials, extend compatibility receipts or replay commands.

This closes the explicit-consent gap and provides lifecycle observability.
Atomic credential replacement, supported renewal sourcing, authenticated
device trust and final licensing/architecture still require further work.

## Independently anchored application trust (7 October)

Application certificate loading now requires a bounded (at most eight
certificates), signature-verified path to the vendor CAs already shipped in
`pybambu/certs/bambu.cert`. The bundle and provisioning reply cannot add
trusted roots. Certificate dates, CA BasicConstraints/path lengths and issuer
keyCertSign usage are enforced; leaf digitalSignature usage is enforced when
present. Unsupported critical extensions are refused rather than silently
ignored. Only the verified path may supply CRL verification issuers, preventing
an unrelated appended certificate from authenticating a fabricated CRL.

Generated tests explicitly select their own independent fixture anchor. The
offline validator offers `--trust-root` for an explicitly chosen independent
anchor, never automatically trusts a bundle root, and defaults to shipped
vendor CAs. Existing vendor CA resources and licenses are unchanged.

This closes the missing application-chain anchoring gap. It does **not** yet
validate the provisioning device chain or bind its leaf to the intended printer;
correlation is still not device authentication. No claim of universal PKIX
profile support is made. No expiry policy or local compatibility review was
extended.
