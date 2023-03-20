# Github Action: Build Msys2 package

An action for makepkg! Automatically sync a branch on your fork with the latest commits from the original repo. Keep things up to date! (It actually works for syncing *any* repo branch with an upstream repo, but I was thinking about forks when I made it.)

<a href="https://www.buymeacoffee.com/aormsby" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-green.png" alt="Buy Me A Coffee" style="height: 51px !important;width: 217px !important;" ></a>

## Intended Workflow

This action is primarily intended to keep **untouched** branches up to date with a remote repo. So let's say you want to keep `main` updated with the upstream repo. You make a new branch `dev` (or whatever) and commit your new changes *there* - not on `main`. If you commit on `main`, you create a split history between your downstream and upstream changes, and any merge from upstream will fail. (This is just how git works.) But if you keep them separate and keep `main` clean, you can always pull the upstream's latest from `main`.

From there, you can make controlled merges into your active branches, make clean PRs directly from `main`, and pretty much do whatever you want with your smoothly updated branch!

<img src="img/workflow.png" alt="sample git workflow">

(Pretend this says 'main' so I don't have to edit the image. Thanks.))

If you *really* want to make commits and merge from upstream on the same branch, I added some notes to achieve this below. This is not recommended, however.

## How to Use

As with any Github Action, you must include it in a workflow for your repo to run. Place the workflow at `.github/workflows/my-sync-workflow.yaml` in the default repo branch (required for scheduled jobs at this time). For more help, see [Github Actions documentation](https://docs.github.com/en/actions).

### Input Variables

| Name                | Required?           | Default           | Example |
| ------------------- |:------------------: | ----------------- | ----------
| rcloneConfig        | :white_check_mark:  |                   | '[onedrive]type = onedrive...'  |
| pgpKey              | :white_check_mark:  |                   | '-----BEGIN PGP PRIVATE KEY BLOCK-----...' |
| pgpKeyPassword      | :white_check_mark:  |                   | 'password' |
| pacmanRepo          |                     |                   | 'msys'             |
| deployPath          |                     |                   | 'onedrive:mirrors/msys2'             |
| customRepos         |                     |                   | '[msys]https://efiles.cf/msys2/$repo/$arch/' |

##### Git Config Settings

Some basic git config settings must be in place to pull and push data during the action process. The following input values are set by default:

- `git_user`
- `git_email`
- `git_pull_rebase_config` -> set the same as git default - 'pull.rebase = false'

These values are reset at the end of the action step. They can be modified as input if you want a different config. **Set any of these config values to `null` to use existing git config data.**

##### When You Want To Merge Into An Acive Working Branch (Not Recommended)

**If there are no conflicts between upstream and downstream branches** => set `git_pull_args: '--allow-unrelated-histories'`.

This will allow divergent histories to be merged, but be careful with the results. It's very likely there will be conflicts in any merge unless you know *for sure* there won't be, so your mileage may vary here.

**If you need to overwrite data for some reason** => `git_pull_args: '-s recursive -Xtheirs'` (or `xOurs`)

Again, not recommended. But if you don't care about losing some data, just choose one side or the other. I actually haven't tested this, but I think it should go through. *Do you feel lucky, punk?*

### Output Variables

**status** - Success when packages were build successfully, otherwise Failed.
**message** - Build message.
**logs** - Build log archive.

## Sample Workflow
This workflow is currently in use in some of my msys2 repos. 

```yaml
on:
  schedule:
    - cron:  '0 7 * * 1,4'
    # scheduled at 07:00 every Monday and Thursday

  workflow_dispatch:  # click the button on Github repo!

jobs:
  sync_with_upstream:
    runs-on: windows-latest
    name: Build Msys2 Package
    steps:
    # Step 1: run a standard checkout action, provided by github
    - name: Checkout main
      uses: actions/checkout@v3
      with:
        ref: main
        # submodules: 'recursive'     ### may be needed in your situation

    # Step 2: run this sync action - specify the rclone config, PGP key & password, and the path to deploy package.
    - name: Run on Msys2
      id: runcmd
      uses: atomlong/run-on-msys-action@master
      with:
        rcloneConfig: ${{ secrets.RCLONE_CONF }}
        pgpKey: ${{ secrets.PGP_KEY }}
        pgpKeyPassword: ${{ secrets.PGP_KEY_PASSWD }}
        pacmanRepo: ${{ secrets.PACMAN_REPO }}
        deployPath: ${{ secrets.DEPLOY_PATH }}
        customRepos: ${{ secrets.CUSTOM_REPOS }}

    # Step 3: Display a message if 'runcmd' step sucess
    - name: Check for new commits
      if: steps.runcmd.outputs.status == Success
      run: echo "The package was build successfully."

    # Step 4: Print a helpful timestamp for your records (not required, just nice)
    - name: Timestamp
      run: date
```
