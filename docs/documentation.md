# GitHub Documentation

- Automate documentation
  - Ensure documents stay up to date by publishing changes with each workflow run.
- Update tickets
  - Once a release succeeds, automatically close tickets associated with the feature branch.
- Send email/marketing content
  - Trigger an email service to send emails notifying stakeholders of updates.
- Run analytics on process
  - Track bottlenecks in the workflow over time to identify areas to refactor.
- Publish messages to queue
  - Publish general purpose messages to a queue so subscribers can create custom responses.
- Anything else...
  - GitHub Actions lets you execute pretty much any automated/scripted action (let your imagination run wild).

Let's deploy documentation to GitHub Pages. For this one refere to [lesson 3](https://github.com/davidainslie/github-actions-deep-dive-lesson-3).

GitHub Pages provides free hosting for your site (a bit like an AWS S3 bucket).
- Any static site can be deployed
  - Plain HTML
  - JS framework
  - and tools such as Jekyll
- Simple to configure
  - Just designate a branch (usually names `gh-pages`) and point to an `index.html` file

Some GitHub Pages use cases:
- Documentation
  - Frameworks like MkDocs or Sphinx can streamline the process
- Testing result output
  - Some testing tools can output HTML, which you can make into a slick dashboard
- Project roadmap
  - A public roadmap for users to see what features are on the horizon

In lesson3 we have the `mkdocs.yaml`:
```yaml
site_name: My Docs
nav:
  - Home: index.md
  - About: about.md 
theme: readthedocs
```

And in the workflow we add the following job:
```yaml
docs:
  runs-on: ubuntu-latest
  needs: deploy
  
  steps:
    - name: Check out code
      uses: actions/checkout@v2
      
    - name: Deploy docs
      uses: mhausenblas/mkdocs-deploy-gh-pages@master
      
      env:
        GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
        CONFIG_FILE: mkdocs.yaml
```

Once run on GHA, go to `settings > pages`.
Then under `Source` select branch `gh-pages` and save where you will see the banner `Your site is published at https://.....`.

We can make sure documentation is kept up to date per release, by altering the workflow's `on push` e.g.
```yaml
on:
  push:
    paths:
      - userguide.md
```