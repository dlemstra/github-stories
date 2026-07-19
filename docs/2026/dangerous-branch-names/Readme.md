# Dangerous branch names

In August 2025, someone from the European Commission reached out to me to ask the ImageMagick project to join a sponsored bug bounty program they were running. Together with [yeswehack](https://www.yeswehack.com/), they created a bug bounty program focused on open source projects that are widely used in the European public sector. Our program has been discontinued because we got so many invalid reports (around 66%) that took up a lot of our time. But because of this, I have done a lot of research on attacks through GitHub Actions workflows, and I decided to look for vulnerabilities in some of the other projects that were selected to start a sponsored bug bounty program. I found a not-so-obvious vulnerability that would expose a secret used in one of those projects. In this blog post, I will explain the vulnerability and how I reported it to the maintainers.

### The project

One of the selected projects was [OpenProject](https://www.openproject.org/), an open source project management platform. The project has a GitHub repository [here](https://github.com/opf/openproject). I started looking at the GitHub Actions workflows defined in the repository and found one that caught my attention. The workflow was defined in the file `.github/workflows/downstream-ci.yml` and had the following content:


```yaml
  pull_request_target:
    types: [opened, reopened, synchronize]
```

This caught my attention because the `pull_request_target` trigger is known to have security implications if not used carefully. I could not find a problem right away because it did not run untrusted code directly. But after looking at the jobs defined in the workflow, I found something interesting.

### The vulnerability

The workflow had the following job defined (trimmed for clarity):

{% raw %}
```yaml
jobs:
  trigger_saas_tests:
    steps:
      - name: Trigger SaaS Tests
        env:
          REPOSITORY: opf/saas-openproject
          TOKEN: ${{ secrets.OPENPROJECT_CI_TOKEN }}
        run: |
          if [ "${{ github.event_name }}" = "pull_request_target" ]; then
            echo The workflow was triggered by a pull request targeting the dev or a release branch
            REF="${{ github.base_ref }}"
            CORE_REF="${{ github.event.pull_request.head.ref }}"
          fi

          echo "Triggering $WORKFLOW_ID workflow on $REPOSITORY branch '$REF', which will check out core branch '$CORE_REF'"
```
{% endraw %}

This job triggers another workflow in a different private repository. The important part is that it uses a secret called `OPENPROJECT_CI_TOKEN` to authenticate the request. I found a way to extract the value of that secret. This workflow directly puts `{% raw %}${{ github.event.pull_request.head.ref }}{% endraw %}` into the `CORE_REF` variable. This means that if I create a pull request from a forked repository, I can control the value of that variable. So I decided to fork this repository to a private repository to see what I could do. If I created a branch with the name `${TOKEN}`, I could print the value of the secret in the logs of the triggered workflow. But because GitHub filters out secrets from the logs, I would not see the actual value of the secret. However, I could use a different approach to extract the value of the secret. I could create a branch with the name `${TOKEN##g}`, and this would remove the first `g` character of the secret when used in the workflow. Because a GitHub token always starts with a `g`, this would allow me to print the secret without the first character. To demonstrate this, I added a secret to my private repository with the name `TOKEN` and the value `github_token_that_starts_with_a_g`. Then I created a branch with the name `${TOKEN##g}` and created a pull request from that branch to the `dev` branch in my private fork. When the workflow was triggered, it printed the following output:

```
Triggering test-saas.yml workflow on opf/saas-openproject branch 'dev', which will check out core branch 'ithub_token_that_starts_with_a_g'
```

I don't know the exact permissions of the token. It might give me access to private information in that repository. But I did not explore that further because doing so would require me to expose the secret in a public workflow run. I reported the issue instead.

### Reporting the vulnerability

Because I only had a Manager account, I had to create a Hunter account to report the vulnerability. The process was easy, but I had to provide a lot of personal information before I was able to create the report. On HackerOne, this is only required once you have been awarded a bounty. I personally think that is a much better approach because it allows security researchers to report vulnerabilities without having to provide a lot of personal information. But given the impact, I decided to provide that information so I could report the vulnerability. I created a report and submitted it to the OpenProject team. After a while, they responded and told me that another security researcher had already reported the same issue. They thanked me for my report and even awarded me a bounty for my duplicate report. This is different from other places like HackerOne, where duplicate reports are not rewarded. This is nice, but I would trade it for not having to provide personal information before being able to report a vulnerability.

### How to fix the workflow

The workflow can be fixed by treating `github.event.pull_request.head.ref` as untrusted input and never injecting it directly into shell commands. Instead, pass it through an environment variable and only reference that variable in quoted form, so the branch name is handled as data and not as executable syntax. It is also safer to avoid `pull_request_target` for jobs that use secrets, unless there is a strong reason to use it, and to reduce token permissions to the minimum required.

For the workflow above, a safer version would look like this:

{% raw %}
```yaml
jobs:
  trigger_saas_tests:
    steps:
      - name: Trigger SaaS Tests
        env:
          REPOSITORY: opf/saas-openproject
          TOKEN: ${{ secrets.OPENPROJECT_CI_TOKEN }}
          CORE_REF: ${{ github.event.pull_request.head.ref }}
        run: |
          if [ "${{ github.event_name }}" = "pull_request_target" ]; then
            REF="${{ github.base_ref }}"
          fi

          echo "Triggering $WORKFLOW_ID workflow on $REPOSITORY branch '$REF', which will check out core branch '$CORE_REF'"
```
{% endraw %}

An extra layer of security could be to only allow pull requests from members of the organization who can create branches in the repository. This would prevent pull requests from forks of the repository, which is where the vulnerability comes from. That can be done by adding the following to the workflow:

```yaml
  trigger_saas_tests:
    if: github.repository == 'opf/openproject'
```

### Closing thoughts

This issue is a good reminder that branch names and other pull request metadata should always be treated as untrusted input. A small workflow detail can turn into a secret exposure when privileged tokens are involved, especially when `pull_request_target` is combined with tokens that can reach private repositories.

The fix in this case is not complicated, but it does require being deliberate: keep untrusted values as data, avoid direct shell injection, and minimize token permissions. If you maintain GitHub Actions workflows, it is worth reviewing every place where pull request data flows into commands or downstream workflow calls. That kind of quick audit is often enough to catch this class of issue before someone else does.

</[@dlemstra](https://github.com/dlemstra)>
