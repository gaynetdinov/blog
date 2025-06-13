---
layout: post
title: "How we deploy Elixir apps"
author: Damir Gainetdinov
date: 2016-06-14 11:13
---

We at adjust recently started to use Elixir. We built a couple of small services using the Phoenix framework which successfully went live. In this blogpost I'd like to talk about, I'd say, the most undiscussed topic when it comes to Elixir — deployment.

<!--more-->

Although you can find some blog posts about deploying Elixir applications, usually after reading them, it still remains unclear how to get the desired command which would deploy your code to production - and which would automate all the routines.

## Capistrano way

The first thing we've tried was `mina`. I'd say, trying to use `Capistrano` or `Mina` is an obvious choice if you come from the Ruby world. However, it becomes clear very quickly that the Capistrano way doesn't fit well for Elixir apps. As you probably know, the preferred way to deploy Elixir applications is to use releases, which means you need a place where a release should be built. It's possible to write a `Capistrano` or `Mina` recipe to clone a project to the production host and build the release there, but that wouldn't be very good idea. Compiling and building a release will take some resources (especially memory) which you don't want to share on production. 

Another option would be to build a release locally using the cross-compiling feature and copy it to production. There are a few gotchas with such approach:  

- there might be some differences in environment (dependency versions, elixir version, etc) between different developers' computers, so two developers might build two different builds based on the same codebase;
- it would be quite tricky to write such a recipe for `Capistrano` (although much easier for `Mina`); generally, using Capistrano just to copy one tarball to a server, unpack it and start it looks like overkill.

So using releases means that there should be a machine where every developer can build a release. Right, a build server! And the problem is that the concept of a build server isn't something familiar for `Capistrano` or `Mina`. So there should be a tool which is aware of the concept of a build server, which maybe even knows how to work with Elixir releases... 

Thankfully such a tool does indeed exist.

## Edeliver

[Edeliver](https://github.com/boldpoker/edeliver) is a deployment tool for Elixir and Erlang projects. It knows how to work with releases and how to apply hot-upgrades, it's aware of a build host and helps you to automate the deployment workflow. `Edeliver` has very good and comprehensive documentation, including several wiki pages describing some edge cases as well. I don't want to review `edeliver`s README in this blogpost, but rather I'd like to cover some of those edge cases and gotchas which we've discovered while using it.

### Auto Versioning

There is a small issue with release names — they must be unique, so every time the `mix edeliver build release` command finishes, a unique release should be generated. `Edeliver` solves this issue by having a special config parameter with which it's possible to append a Git revision, Git branch, build date, etc to a release name. So you don't need to go to the `mix.exs` file and change `version` in `project/0` function – `edeliver` does it for you. We found that `AUTO_VERSION=git-branch+git-revision` generates sufficiently unique release names. With this combination a release name would be something like "awesome_adjust_app_0.0.1+master-01b4601.release.tar.gz".

### Custom environments

By default `edeliver` provides only two environments to which it's possible to deploy — staging and production. There is no easy way to add custom environments, but as it turned out it's still possible to achieve that by overriding `STAGING_HOSTS` and `STAGING_USER` variables in `.deliver/config`. 

Let's say we want to add `beta` and `qa` environments. To do so `.deliver/config` should look like this:

```sh
QA_USER="qa_user"
BETA_USER="beta_user"

QA_TEST_AT="/qa/path/where/to/deploy/"
BETA_TEST_AT="/beta/path/where/to/deploy/"

QA_HOST="qa.company.com"
BETA_HOST="beta.company.com"

QA_NODES="${QA_USER}@${QA_HOST}:${QA_TEST_AT}"
BETA_NODES="${BETA_USER}@${BETA_HOST}:${BETA_TEST_AT}"

if [[ "$DEPLOY_ENVIRONMENT" = "qa" ]]; then
  TEST_AT="${QA_TEST_AT}"
  STAGING_HOSTS="${QA_HOST}"
  STAGING_USER="${QA_USER}"
elif [[ "$DEPLOY_ENVIRONMENT" = "beta" ]]; then
  TEST_AT="${BETA_TEST_AT}"
  STAGING_HOSTS="${BETA_HOST}"
  STAGING_USER="${BETA_USER}"
elif [[ "$DEPLOY_ENVIRONMENT" = "staging" ]]; then
  TEST_AT="/staging/path/where/to/deploy"
  STAGING_HOSTS="staging.company.com"
  STAGING_USER="staging_user"
fi
```

As you can see, the `ENVNAME_NODES` variables should be added and then based on `$DEPLOY_ENVIRONMENT`, staging related variables should be overridden.
 
Also, it's important to add the `.deliver/help` file where these new environments should be added:

```sh
#!/usr/bin/env bash

print_custom_commands_help() {
  echo -e "  ${txtbld}Custom Deploy Environments:${txtrst}
  edeliver deploy release|upgrade [to] qa|beta|staging|production
  "
}

accepts_custom_command_argument() {
  local _command="$1"
  local _argument="$2"
  case "${_command}" in
    (deploy)
      case "${_argument}" in
        (qa|beta|staging|production)
          DEPLOY_ENVIRONMENT="${_argument}"
          return 0 ;;
        ("") return 0 ;; # default: 'staging'
        (*)  return 1 ;; # unknown deploy environment
      esac ;;
    (*) return 1 ;; # unknown custom command
  esac
}
```

With this config, it would be possible to deploy a release to the `beta` and `qa` hosts (in addition to `staging` and `production`) and to maintain these custom hosts. For example, in order to check the version of the `beta` host, you'd run a command like this: `mix edeliver version beta`. 

### Deploy notifications

It's quite common to send notifications about successful deployments. For example, we might display such notifications in a Slack channel. `edeliver` has hooks which can be implemented as bash functions. For example, there are two hook functions: `pre_upgrade_release()` and `post_upgrade_release()`. They are called exactly before applying an `upgrade` and right after an `upgrade` has been applied, respectively. Notifications about deployment usually contain information about the person who deployed, the Git branch and revision, and the environment name (staging/production). 

The issue here is that you can't get a Git branch and Git revision out of a release since a release is just a binary. With Capistrano, you can just run a couple of git commands on the target host to get the necessary data. With `edeliver` it becomes a bit more tricky. The current workaround we use is to include the Git revision and Git branch into a release name using the following config: `AUTO_VERSION=git-branch+git-revision`. This is as I described in the previous section on Auto-Versioning. Then in the project itself a `Notifier` module might look as follows:

```elixir
defmodule MyApp.Notifier do
  def notify(username, env, event) do
    {branch, revision} =
      :my_app
      |> Edeliver.release_version
      |> to_string
      |> extract_git_info

    build_notification(event, username, hostname, revision, branch, env)
    |> send_notification
  end

  defp extract_git_info(release_version) do
    [_, branch_revision] = String.split(release_version, "+")
    list = String.split(branch_revision, "-")
    rev = List.last(list)
    branch = List.delete_at(list, -1) |> Enum.join("-")

    {branch, rev}
  end

  defp build_notification(event, username, hostname, revision, branch, env) do
     # create a map/list/keyword with necessary data for a notification
  end

  defp send_notification(notification) do
    # send a HTTP request, create a job, etc  
  end

  defp hostname do
    System.cmd("hostname", []) |> elem(0) |> String.strip
  end
end
```

Then the `pre_upgrade_release()` and `post_upgrade_release()` hooks might look like this:

```sh
pre_upgrade_release() {
  status "Sending 'deploying' notification"
  send_deploy_notification "deploying"
}

post_upgrade_release() {
  status "Sending 'deployed' notification"
  send_deploy_notification "deployed"
}

send_deploy_notification() {
  local _event="$1"
  local _username=$(git config user.name)

  __sync_remote "
    [ -f ~/.profile ] && source ~/.profile
    set -e
    cd '$DELIVER_TO/$APP'

    ./bin/$APP rpc Elixir.MyApp.Notifier notify \"[<<\\\"$_username\\\">>, <<\\\"$DEPLOY_ENVIRONMENT\\\">>, <<\\\"$_event\\\">>].\" >/dev/null
  "
}
```

However, there are two flaws here. First, it works only when applying upgrades - not for releases. And second, when calling `Elixir.MyApp.Notifier` from `pre_upgrade_release`, `Edeliver.release_version` returns a git revision of the currently deployed release. So 'deploying' notification would have a git revision of the currently deployed version and the 'deployed' notification would have a git revision of the new version.

### Different configurations on different deploy hosts

Most probably, your application has different settings for staging and production environments. Which means that you need either to build a release for each environment separately or somehow provide different settings on different hosts for the same release. `Edeliver`, following a philosophy "build once, deploy everywhere" suggests to solve this problem by using `LINK_SYS_CONFIG` or `LINK_VM_ARGS` config variables as described on [this wiki page](https://github.com/boldpoker/edeliver/wiki/Use-per-host-configuration). 

I'll describe briefly how it works with `LINK_VM_ARGS` variable. The logic is the same for `LINK_SYS_CONFIG`. So it works as follows: you need to create a file which should have the same path on both `staging` and `production` hosts with config values specific for the target host. This could be `/home/deploy_user/my_app/vm.args`, for example. Then in `.deliver/config` you can specify `LINK_VM_ARGS=/home/deploy_user/my_app/vm.args`. 

When making a release or an upgrade, `edeliver` would put a symlink inside a release (instead of the real generated `vm.args`) which will point to `/home/deploy_user/my_app/vm.args`. So this tricky and sophisticated approach solves the issue. In theory. I [couldn't actually make it work](https://github.com/boldpoker/edeliver/issues/87#issuecomment-221803959). After a release deployment I see a symlink as expected, but on release start, my custom symlinked `vm.args` file should replace `vm.args` from `running-config` which does not happen. However, if I remove the `running-config` folder first and start a release afterwards, it works. 

So since this approach didn't fully work, we decided to build a release per environment, which is also suboptimal: 

* you need to build a release per environment
* it violates a release philosophy: build once, deploy everywhere
* error prone: somebody can by mistake deploy a release on production, which has been built for staging

To partially fix the last bullet from the list above it's possible to add a `mix-env` parameter to the `AUTO_VERSION` config value: `AUTO_VERSION=git-branch+git-revision+mix-env`. So every build would have `-environment` in its name to indicate for which environment a release has been built. 

Usually, for Phoenix applications secret production settings (like database connection credentials for production DB) are stored in `prod.secret.exs`. This file is not under version control, but it should be inside a release. To achieve that you might want to put this file manually into the build host, but the issue here is that a folder where a project is built is cleaned by `edeliver` before every release build. The 'cleaning' means that everything which is not under version control will be removed before every build, so `config/prod.secret.exs` will be gone. To avoid that there is an option to explicitly instruct `edeliver` which folders should be cleaned. Having the config option `GIT_CLEAN_PATHS="_build rel deps"` tells `edeliver` to clean `_build`, `rel` and `deps` folders before every release build, so `config` folder stays untouched and therefore `prod.secret.exs` stays alive between release builds.

### Bonus: Change font color output

For light terminal themes `edeliver` output by default looks as follows:

![screenshot](/assets/images/edeliver_solarized_light.png)

There is an option to change that by overriding the color of the font:

```sh
# Reassigning output font colors, otherwise some text is not visible in solarized light theme.
bldwht=${txtbld}${txtblue}
txtwht=$(tput setaf 2)
```

With the fix the output looks as follows:

![screenshot](/assets/images/edeliver_solarized_fixed.png)


### Alternatives

Currently, there are not so many alternatives to `edeliver`. But there is at least one: [dicon](https://github.com/lexmag/dicon). It's in the early stages of development, it doesn't have comprehensive readme, it's not aware of build host and it does not support hot-upgrades yet. However, `Digital Conveyor` has some niceties: it's written completely in Elixir, it's small and it supports configurations per target host out of the box. It will be interesting to see how `dicon` will be evolving.

## Conclusion

`Edeliver` is a great, ready-to-use deployment tool packed with a lot of useful features. It works with releases, supports hot-upgrades and build host concept, has very good documentation and gives you simple commands to automate deployment routines. Importantly, the project is in active development. I'd like to thank [bharendt](https://github.com/bharendt) for amazing support, almost every tip or trick I've described in the post is a result of a detailed answer from him to an opened issue. Sometimes I had a feeling that I'm literally chatting with him in the `Issues` tab, that's amazing.

That's it for today. Happy deploying!
