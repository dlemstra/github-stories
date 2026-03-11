# How I could have compromised several millions of installations of a popular editor extension

I recently learned how I can secure the actions in my GitHub repositories to prevent attackers from compromising my open source projects. While securing the ImageMagick project and my own repositories I wondered if other popular open source projects were also secured. So I decided to check some projects on GitHub to see if they were secured. And I found a popular editor extension that was not secured. That extension has several millions of installations. In this story I will explain how I secured my GitHub actions and how I could have compromised this extension.

### Limiting the permissions of GitHub actions

The first step to secure my GitHub actions was to limit the permissions of the actions. Old projects often have actions that run with more permissions than needed. Most of my projects ran with the following permissions:

```
GITHUB_TOKEN Permissions
  Actions: write
  Attestations: write
  Checks: write
  Contents: write
  Deployments: write
  Discussions: write
  Issues: write
  Metadata: read
  Models: read
  Packages: write
  Pages: write
  PullRequests: write
  RepositoryProjects: write
  SecurityEvents: write
  Statuses: write
```

While recently created projects run with the following permissions:

```
GITHUB_TOKEN Permissions
  Contents: read
  Metadata: read
  Packages: read
```

As you can see the old projects have many more permissions than needed. So I changed the permissions of my old projects and limited them to only the permissions needed for the actions to run. For most of my projects this meant changing the permissions to the following:

```yaml
on:
  push:
    branches:
    - main

permissions:
  contents: read
```

This change ensures that the actions can only read the contents of the repository and nothing else. This prevents attackers from using the actions to modify the repository when they manage to execute something in that workflow. But during the training I also learned about `pull_request_target`.

### What is `pull_request_target` and why is it dangerous?

Besides a trigger for `pull_request` there is also a trigger for `pull_request_target`.

```yaml
on:
  pull_request_target:
    branches:
    - main
```
The difference between these two triggers is that `pull_request` runs the action in the context of the fork while `pull_request_target` runs the action in the context of the target branch. This means that when you use `pull_request_target` the action has access to the secrets of that target repository. This can be dangerous when you use this trigger in combination with untrusted code from a forked repository. An attacker could create a pull request from a forked repository that executes malicious code. And because the action runs in the context of the target repository it has access to all secrets of that project. Luckily I did not use this trigger in any of my projects. But the extension I checked did use this trigger in their workflow.

### Why was the workflow of the extension vulnerable?

The workflow started with the following code:

```yaml
on:
  pull_request_target:
    branches:
      - main
permissions:
  contents: write
  pull-requests: write
```

As you can see the action uses the `pull_request_target` trigger and has write permissions for both contents and pull requests. These permissions were added because of the following job in the workflow:

```yaml
  merge-dependabot:
    name: Merge Dependabot
    runs-on: ubuntu-latest
    needs:
      - check
    if: github.event.pull_request.user.login == 'dependabot[bot]'
    steps:
      - name: Merge Dependabot PRs
        run: gh pr merge
```

This job was added to automatically merge pull requests created by Dependabot. This would not be a problem if this was the only job in the workflow. But there was also a job that built the extension:

{% raw %}
```yaml
  check:
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{github.event.pull_request.head.ref}}
          repository: ${{github.event.pull_request.head.repo.full_name}}

      - name: Setup node
        uses: actions/setup-node@v5

      - name: Install deps
        run: npm ci

      - name: Run check
        run: npm run check
```
{% endraw %}

This job checks out the code from the pull request and builds the extension. This means it uses the code from the forked repository and runs the command `npm run check`. But because it uses the `pull_request_target` trigger this job runs in the security context of the target project with write permissions for both contents and pull requests.

### What could I have done with this workflow?

The command `npm run check` runs a script defined in the `package.json` file of the extension. Because I control the code in the forked repository I could make the following change to the `package.json` file:

```patch
--- a/package.json
+++ b/package.json
-    "check": "eslint --fix --ext .ts .",
+    "check": "./malicious-script.sh",
```

And inside the `malicious-script.sh` file I could have created something that would send the secret that is used to publish the extension to my server and publish my own malicious version of that extension. Or I could modify the code of the extension in the target repository. Because the workflow runs with write permissions for contents I could also have added a backdoor to the code and committed that change back to the repository.

### What did I do instead?

I did not want to test the vulnerability in public so I created a private fork of the extension to see what would happen. I created a pull request in that private repository with the changes explained above and saw that I could execute arbitrary commands in the context of the target branch. Because of the impact of this vulnerability I wanted to report it to the maintainers of the project. I checked the Security tab of the project to see if there was a way to report this vulnerability privately. But this project did not have a SECURITY.md file that explained what I need to do. After a quick online search I found a way to report security vulnerabilities. I reached out to them and explained the vulnerability and how I could have exploited it. They required me to create a proof of concept to validate the vulnerability. I updated my private fork to run a simple echo command instead of the malicious script and that produced the following output in their workflow:

```
> echo 'skipping check'

skipping check
```

This confirmed that I could execute arbitrary commands in the context of the target branch and they applied a fix to their workflow and make their project more secure. I am glad that I could report this vulnerability to the maintainers and that they took it seriously.

### Lessons learned

If you maintain GitHub Actions workflows in your projects, here are some key things to review:

- **Audit your `pull_request_target` workflows**: If you use this trigger, ensure you never checkout or execute untrusted code from the pull request. Only use it when you need access to secrets and only run trusted code from your repository.
- **Apply least privilege permissions**: Limit workflow permissions to only what's absolutely necessary. Start with `contents: read` and only add more permissions when required.
- **Separate sensitive jobs**: If you need `pull_request_target` for specific jobs like auto-merging Dependabot PRs, keep those jobs isolated from any jobs that run untrusted code.
- **Add a SECURITY.md file**: Make it easy for security researchers to report vulnerabilities responsibly by documenting your security disclosure process.
- **Stay informed**: Regularly review GitHub's security best practices and updates to ensure your workflows remain secure against emerging threats.
- **Use code scanning tools**: Leverage GitHub's code scanning features to automatically detect potential security issues in your workflows or use an open source tool lik [zizmor](https://github.com/zizmorcore/zizmor).

When a single pull request could have compromised millions of installations without any maintainer approval, we need to take these vulnerabilities seriously. I encourage all maintainers of popular projects to audit their workflows and apply these security best practices. The supply chain security of the entire ecosystem depends on it.

</[@dlemstra](https://github.com/dlemstra)>
