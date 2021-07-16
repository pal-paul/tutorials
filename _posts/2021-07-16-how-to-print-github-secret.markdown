---
layout: post
title: "How to print Github secret in Action"
date: 2021-07-16 12:12:12 +0200
categories: Github Action
permalink: /:title/
---

In some scenario we may need to print or debug Github action secret, but the problem is that we can’t debug secrets. If we try to debug print or secrets we’ll get `***` in the log.

```
- name: print-secret
  run: echo ${{secrets.YOUR_SECRET_NAME }}
```

From the above the `echo` command, the Github action secret will show as `***` in the log.

```
Run echo ***
```

This is makes sense because Github is trying to help us to keep the secrets secret as the action log is visible for all having access to the repository.

There is a way to show this secret if we really want to print or debug and show it log to verify.

We can separate the characters with a space using the following code. The secret will visible in the action log.

```
- name: print-secret
  run: echo ${{secrets.YOUR_SECRET_NAME }} | sed 's/./& /g'
```

From the above the `echo` command, the Github action secret will show as actual secret in the log but seperated with space in each char.

```
Run echo *** | sed 's/./& /g'
t e s t
```

_Note: Its recommended that we should not print any secret in the github action log, once we debug or verified the secret in log we should remove the print option before merge._
