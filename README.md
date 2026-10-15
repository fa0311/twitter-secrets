# twitter-secrets

Shared, sops-encrypted credentials for the Twitter/X ecosystem CI jobs
(TwitterInternalAPIDocument, twitter-openapi, x-client-transaction-id-pair-dict, ...).
Rotate cookies or proxies here once instead of in every repository's secrets.

## Contents

| file | what |
| --- | --- |
| `secrets.enc.yaml` | `cookie_json` (X session cookies, browser extension JSON format) and `proxy_list` (one proxy URL per line, e.g. `socks5h://user:pass@host:1080`; legacy `host:port:user:pass` means HTTP) |
| `keys-ci-age.txt.gpg` | the CI age private key, encrypted to the maintainer's PGP key |
| `.sops.yaml` | recipients: CI age key + maintainer PGP key `EAB5AF5AA7ED7A16C4027B5283A8A5E74872A8AA` |
| `action.yml` | composite action that decrypts everything inside a workflow |

## Use from GitHub Actions

Add the age private key as the `SOPS_AGE_KEY` secret of the consuming repository, then:

```yaml
- uses: fa0311/twitter-secrets@main
  with:
    age-key: ${{ secrets.SOPS_AGE_KEY }}
    cookie-path: cookie.json # default
# cookie.json is written to the workspace and PROXY_LIST is exported as an env var
```

## Edit locally

```sh
export GNUPGHOME=~/.gnupg            # the maintainer PGP key decrypts without the age key
sops secrets.enc.yaml                # edit in $EDITOR, re-encrypted on save
sops decrypt --extract '["proxy_list"]' secrets.enc.yaml
```

To import a fresh browser cookie export:

```sh
uv run --with pyyaml python - <<'PY'
import json, subprocess
cookies = [c for c in json.load(open("cookie.json")) if c["domain"].endswith("x.com")]
minimal = json.dumps([{"name": c["name"], "value": c["value"], "domain": c["domain"]} for c in cookies])
subprocess.run(["sops", "set", "secrets.enc.yaml", '["cookie_json"]', json.dumps(minimal)], check=True)
PY
```

## Rotate the CI key

```sh
age-keygen -o ci.txt                       # new key pair
sed -i "s/age1[0-9a-z]*/$(grep -o 'age1[0-9a-z]*' ci.txt)/" .sops.yaml
sops updatekeys secrets.enc.yaml
gpg -e -r EAB5AF5AA7ED7A16C4027B5283A8A5E74872A8AA -o keys-ci-age.txt.gpg ci.txt
gh secret set SOPS_AGE_KEY -R fa0311/<repo> < ci.txt   # for every consuming repo
```

## As a submodule

`git submodule add https://github.com/fa0311/twitter-secrets` also works for local tooling;
decrypt with `SOPS_AGE_KEY_FILE=... sops decrypt twitter-secrets/secrets.enc.yaml`.
