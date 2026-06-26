# Publish Checklist

Use this directory as the publishable copy.

## Safe to publish

- `_worker.js`
- `README.md`
- `LICENSE`
- `CHANGELOG`
- `MODIFICATIONS.md`
- `wrangler.toml` after filling your own values

## Do not publish private values

- Cloudflare account id
- Real KV namespace id
- Worker secret values such as `ADMIN`, `PASSWORD`, `KEY`, or `UUID`
- Private subscription token
- Private custom domain if you do not want it public

## Recommended GitHub flow

1. Fork `cmliu/edgetunnel`.
2. Copy `_worker.js` from this directory into the fork.
3. Copy `MODIFICATIONS.md` if you want to document the behavior.
4. Keep the upstream `LICENSE`.
5. Do not commit a real `wrangler.toml` with private ids.

For a pull request, use `../edgetunnel-ech-fix.patch` as the minimal patch.

The upstream `.github` workflows are intentionally not included in this publishable copy to avoid automatic upstream sync changing your modified branch unexpectedly.
