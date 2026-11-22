## What and why

<!-- What does this PR change, and why is it needed? -->

Closes #

## Type of change

- [ ] Feature
- [ ] Bug fix
- [ ] Refactor (no behaviour change)
- [ ] Chore (dependencies, config, tooling)
- [ ] Docs
- [ ] Infrastructure (Terraform / Helm)

## How to test

<!-- Steps, commands, URLs or sample requests a reviewer can use to check this. -->

1.

## Screenshots

<!-- For UI changes. Delete this section if not needed. -->

## Related PRs

<!-- Links to PRs in other repos that go with this one, e.g. the matching change in spring-api / express-api. -->

## Checklist

- [ ] Branch is up to date with `main`
- [ ] Local checks pass (see [Contributing](https://github.com/medica-health#5-check-your-work-locally))
- [ ] API changes are made in **both** `spring-api` and `express-api`, or there's an issue for the other one
- [ ] New or renamed env vars are added to `.env.example`, `docker-compose.yml`, and the Helm chart / Terraform secrets
- [ ] No secrets, `.env`, `*.tfvars` or Terraform state are committed
- [ ] Infrastructure PRs: `terraform plan` / `helm template` output is pasted below, with secrets removed

<details>
<summary>Plan / diff output (infrastructure only)</summary>

```

```

</details>
