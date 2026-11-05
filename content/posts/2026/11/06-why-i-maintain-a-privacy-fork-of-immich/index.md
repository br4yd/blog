+++
date = '2026-11-06T00:41:08+01:00'
draft = false
title = 'Why I maintain a privacy fork of Immich'

tags = ['Immich', 'Self-Hosting', 'Privacy', 'Linux']

[cover]
image = '2026-11-06_immich-post-header.png'
alt = 'A screenshot of the Immich website with the headline Self-hosted photo and video management solution, above a preview of the Immich photo timeline.'
+++

[Immich](https://immich.app) is a great self-hosted alternative to Google Photos and I really like it. But there is one bug that bothered me enough to fix it myself. The fix did not get merged, so I now maintain a small [fork](https://github.com/br4yd/immich) with its own Docker image. This post explains what the bug is, why the fix is not in Immich itself and how you can use the fork if the bug bothers you too.

## The problem
Immich has a server setting called **Public Users**. According to its description, the user list is only available to admins when the setting is disabled. In the web app this works as expected.

The mobile app is a different story. Since Sync v2, the server sends the user list to the app through a sync feed called `UserV1`, and that feed sent every user to every client. It did not check the setting or whether the person asking is an admin. So even with Public Users disabled, anyone with an account could open Partner Sharing or Album Sharing in the app and see the name and email address of every other user on the server.

This is not a problem if you only share your server with your family. On a server with people who don't know each other, it is a real privacy issue, especially because the setting tells admins that the list is hidden.

The bug was [first reported in December 2025](https://github.com/immich-app/immich/issues/24528). I ran into it myself and [reported it](https://github.com/immich-app/immich/issues/30250) in July 2026, which was closed as a duplicate.

## The fix
The fix has two parts:

1. **Only sync the users you are allowed to see.** If Public Users is disabled, non-admins now only receive themselves, their partners and people they share an album with. Existing shares keep displaying correctly and admins still get everyone.
2. **Clean up devices that already have the full list.** The sync protocol only adds or updates users. It never removes the ones that quietly became invisible, so a device would keep its cached list until you log out. To solve this, all non-admin sessions now do a full re-sync whenever the setting changes, in either direction.

The second part also taught me something. My first version of the re-sync handler never ran, because I had scoped it to the microservices worker. The `ConfigUpdate` event it listens to only fires on the API worker, where the settings get changed. It's a small mistake, but without fixing it the whole cleanup would have silently done nothing.

## Why it is not in Immich
I opened a [pull request](https://github.com/immich-app/immich/pull/30564) in August 2026. The review was friendly and started with:

> The code in this PR is very clear and looks good.

It was closed anyway. The maintainer's reasoning is that the Public Users setting is a band-aid for a bigger problem. They would rather build something like Google Photos has, where every account gets its own group of known users that grows as you interact with people. With that in place, the sync would simply use this group instead of the whole user table.

I asked if it would make sense to merge my version until then. The answer was that the team does not really want to keep maintaining and fixing this feature, and:

> If any work is going to be done here it would probably be to remove the feature entirely until it can be properly implemented.

I would have liked to see it merged, but I understand that decision. Every feature in a project of this size is something the team has to maintain for a long time, and they are the ones who have to do that. It's their call to make.

It does leave me with a problem though, because the issue is still open and I still want the setting to do what it says. So I forked it.

## How the fork works
The fork is deliberately as small as possible. It consists of seven commits on top of each official release: four for the fix and three for the CI workflow that builds the image. Every release gets its own `deploy/vX.Y.Z` branch, based on the upstream tag, and pushing a `vX.Y.Z-privacy-fix` tag builds the image and publishes it to the GitHub Container Registry.

I only build the server image, since the fix doesn't touch anything else. The machine learning image, the web app and the mobile apps stay the same as upstream, and there are no database migrations.

The first image was v3.1.0 at the end of August, and I have built eight versions so far. For updates I use a small Claude Code skill that moves the commits to the new release and watches the build. A clean cherry-pick doesn't prove that the fix still works, though. Upstream could add a new way of syncing users at any time, so any upstream change to the sync code gets a closer look before an image is published.

## How to use it
In your `docker-compose.yml`, replace only the server image:

```yaml
services:
  immich-server:
    image: ghcr.io/br4yd/immich-server:v3.3.6-privacy-fix
```

Keep the official `immich-machine-learning` image on the matching version. The current version may be newer than the one in this post, so check the [available images](https://github.com/br4yd/immich/pkgs/container/immich-server) first.

> **Important:** The re-sync only runs when the setting changes. If Public Users is already disabled when you switch to this image, devices keep the list they have cached. To clear it, turn the setting on and back off once after switching.

A few other things you should know:

- **Only amd64:** There is no arm64 image at the moment, because building it on GitHub's runners was far too slow.
- **Updates will lag behind:** A new image only exists after I have updated the fork, so you might be a bit behind the official release.
- **You have to trust me:** You would be running a server image built by some person on the internet. All of the changes are public, so please read them, or build the image yourself if you prefer that.
- **Going back is easy:** Without any migrations in the fork, you can switch back to the official image of the same version at any time.

## How long will this exist?
Hopefully not forever. If Immich ships the known users feature or removes the setting, the fork has no reason to exist anymore and I'll happily archive it. Until then I'll keep it up to date for as long as I run Immich myself.

If you run into problems with the fork, please [write me an email](mailto:info@braydmedia.de) and don't report them to the Immich project, because the team has nothing to do with it.

*Header image: screenshot of [immich.app](https://immich.app), not covered by this blog's license.*
