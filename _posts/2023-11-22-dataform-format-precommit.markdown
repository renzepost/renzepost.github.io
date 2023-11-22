---
layout: post
title: "A simple way to format your Dataform project before each Git commit"
date: 2023-11-22 13:00:00 +0200
categories: dataform format git pre-commit
---
If you work with Dataform core in your local environment (instead of using the GCP Cloud IDE), you will want to run `dataform format` before committing your code to neatly format your `.sqlx` and `.js` files. This can be easily automated in Git by creating a pre-commit hook. This short tutorial will show you how to create such a pre-commit hook and share it with the rest of your team. I asked ChatGPT to create this pre-commit hook for me, and it got it almost right! I just had to adjust a few things to make it work correctly. The tutorial assumes you already have the Dataform CLI installed on your machine. If not, please follow the [documentation](https://cloud.google.com/dataform/docs/use-dataform-cli).

# The pre-commit hook
In your Dataform project's root directory, create a new directory called `.githooks` and inside that directory, create a file called `pre-commit`, so it should look like this: `your_dataform_project/.githooks/pre-commit`. Edit the `pre-commit` file and paste the following content:

```sh
#!/bin/sh

# Stash any unstaged changes
git stash -q --keep-index

# Run dataform format
dataform format

# Check if the dataform format command changed any files
CHANGED_FILES=$(git diff --name-only definitions/ includes/)
if [ -n "$CHANGED_FILES" ]; then
    echo "Some files were formatted by dataform. Please add the changes to your commit."
    git stash pop -q
    exit 1
fi

# Reapply the stash to restore unstaged changes
git stash pop -q
```

Save the file. Now you need to make this script executable. In Linux, run the command `chmod +x .githooks/pre-commit`.

This shell script first stashes any unstaged changes, because we only want the hook to fail if `dataform format` modifies any files. After that, it runs `dataform format` and then checks if it changed any files in the `definitions/` and `includes/` directories. If so, it will notify you, reapply the stash and exit with status code 1. If no files were modified, it will reapply the stash and exit normally. 

# Activate the pre-commit hook
Now all we have to do, is to tell Git that the newly created directory `.githooks` contains the Git hooks for this project: `git config --local core.hooksPath .githooks/`.

You can now test your newly created hook. Make some changes to any `.sqlx` or `.js` files in the `definitions/` or `includes/` directories, run `git add` to stage the changes, then run `git commit`. If all goes well, Git will run the shell script before the commit.

# Commit your changes
Make sure that the newly created `.githooks` directory and its contents make it to the main branch of your Dataform project's repository. Now the rest of your team can easily configure the pre-commit hook. All they have to do is to run `git config --local core.hooksPath .githooks/`. You could update your repository's `README.md` with this information.

# Next steps
Now that you have a working pre-commit hook, you should think about adding this check in your CI/CD workflow. With the pre-commit shell script, this should be relatively simple to do. This will ensure that any code committed to your repository is correctly formatted. I won't show you how to do this in this tutorial, because it greatly depends on the automation server your team is using (Jenkins, GitHub Actions etc.)
