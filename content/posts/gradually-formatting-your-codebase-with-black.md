+++
title = "Gradually formatting your codebase with black"
date = "2020-05-25T10:41:49-03:00"
author = "Rocio Aramberri"
keywords = ["black", "black action", "gradual black", "incremental black", "python formatter", "python black"]
tags = ["python"]
+++

I've just created my first Github Action: [https://github.com/rocioar/gradual-black-formatter](https://github.com/rocioar/gradual-black-formatter). It's called "Gradual Black Formatter", and it allows you to gradually format your code using [black](https://github.com/psf/black) via Github Actions.

If your codebase is small and well tested, this Github Action would be of no use to you, as you could just format your code on a single PR. On the other hand, if you have a big enough codebase in which making a big PR with lots of formatting changes is a risk. Then look no further.

## How does it work?

The Github Action just does one thing: runs black on the **x** least recently modified files.

Example action that runs black on the 10 least recently modified files:

```yaml
- name: Apply Black Gradually
  uses: rocioar/gradual-black-formatter@v1
  with:
    number_of_files: 10
    ignore_files_regex: *test*,*migrations*
```

As you can see on the example above, it accepts two parameters:
* `number_of_files` to indicate how many files you want `black` to format each time.
* `ignore_files_regex` to indicate the files you want to ignore, if any. You might want to exclude tests since they are low-risk and you could make a single PR to format all tests at once.

## How can I use it on my project?

To use it, you can combine it with a few other actions to accomplish the following workflow:

1. Checkout the code
2. Apply Black to **x** files
3. Commit the code to a branch
4. Send a PR

Here is an example workflow that uses [actions/checkout@v2](https://github.com/actions/checkout) to checkout the code and [peter-evans/create-pull-request@v2](https://github.com/peter-evans/create-pull-request) to make a commit and create a PR:
```yaml
name: Apply Black Gradually

on:
  pull_request:
    types: [closed]

jobs:
  apply-black:
    if: github.head_ref == 'black'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          ref: master
          fetch-depth: 0
      - name: Apply Black Gradually
        id: black
        uses: rocioar/gradual-black-formatter@v1
        with:
          number_of_files: 3
          ignore_files_regex: *test*,*migrations*
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: Apply black to ${{ steps.black.outputs.number_of_modified_files }} files
          committer: GitHub <noreply@github.com>
          author: ${{ github.actor }} <${{ github.actor }}@users.noreply.github.com>
          title: Apply black to ${{ steps.black.outputs.number_of_modified_files }} files
          body: "Auto-generated PR that applies black to: ${{ steps.black.outputs.modified_file_names }}."
          branch: black
```

**Important Note**

If you want the files to be formatted in order starting from the least recently modified files, make sure you pass `fetch-depth: 0` to the checkout action so that all the revision history is retrieved.

### When will the action be executed?

As configured on the example above the action will be executed every time a PR to the `black` branch is closed.

To start it off, you can make a PR with the workflow to the `black` branch, once you merge it, the process will start:

1. The action will create a PR with the formatted files.
2. You will review the PR and merge it.
3. It will go back to step 1 until all files have been formatted.

On the following image you can see an example of a PR that was created by the action:
{{<figure src="/img/example_pr.png" title="Steve Francia" position="center" style="border-radius: 8px;">}}

## Wrapping up

Formatting your code with `black` will make your codebase more consistent, and help you avoid many derails during code reviews. This Github Action will help you format your code with the least disruption.