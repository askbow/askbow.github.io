---
layout: post
title:  "How to find initial function authors in large Git repo"
date:   2023-12-17 10:30:00 +0100
categories: git
---

Sometimes, when onboarding into a new codebase we need to explore a little. Some older documentation references might include only partial commit hashes. Other methods were touched by several commits that may include additional context. Finally, we may just want to find the initial author of a given method. How to go about those?

>Here and further, I'll be looking at the [azure-quickstart-templates](https://github.com/Azure/azure-quickstart-templates) repo in the state as of this writing in Dec 2023.


## How to find full commit hash from a partial?


Let's find the full hash of a commit starting `82a5218` that we found in some doc from a few years back:

```bash
$ git show "82a5218" --no-patch --pretty="%H %s"
82a5218d94226a85083c9cf748e8549500cdf405 End-to-end Azure ML set up reference implementation (#12006)
```
* `%H` for the full hash message, `%s` for the single-line commit message

Notice that this repository is large enough for git to start using slight longer hashes in default output compared to what we found in the docs:

```bash
$ git show "82a5218" --no-patch
82a5218d9  End-to-end Azure ML set up reference implementation (#12006)
```
* git automatically shortens and extends the displayed hash so that it remains powerful enough for uniqueness, while being as human-readable as possible


How about the parents of that commit? We can use `git rev-parse` with some modification to the hash reference for that:

```bash
$ git rev-parse 82a5218d9^@
f88c1f77c8340cb914d999c7d005b512ad4ab9c6
```
* the `^@` literally means "all parents of the specified commit", or more specifically "anything that is reachable from the commit's parents" excluding the commit itself.

## How to find all commits that happened between two other events?

By events I mean also commits, but in a more general form of "pointers" or refs. These include branch (and `HEAD`) pointers, tags, etc.

For a more concise output, let's count all commits since `82a5218`:

```bash
$ git log 82a5218..HEAD --oneline | wc -l
3249
```
* the `A..B` notation means "from A to B inclusive"

## How to find the very first commit for a specific method?

Let's say we are looking at the origins of the KeyVault usage.


```bash
$ git log -G'Microsoft.KeyVault/vaults\@.+' --oneline | tail -1
e33bf9904 New from bicep example: 101/function-http-trigger (#11759)
```
* [e33bf9904](https://github.com/Azure/azure-quickstart-templates/commit/e33bf9904d599f5b7fd401e8171328d460af2fbc#diff-f8d1aa6307090ce36078ceb0660eef646ab3c4a81380dadb3b93f88a0c2cd1edR180)

Notably, the built-in regex of `git log` allows us to search through commit contents. In this case, we are only interested in the very first instance, which is the last in the log output, hence `tail -1`.

Another form of git built-in regex allows us to find for example the function definitions across the code:

```bash
$ git grep 'resource keyVault' | tail
quickstarts/microsoft.network/azurefirewall-premium/main.bicep:resource keyVault 'Microsoft.KeyVault/vaults@2019-09-01' = {
quickstarts/microsoft.network/azurefirewall-premium/main.bicep:resource keyVaultName_keyVaultCASecret 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = {
quickstarts/microsoft.storage/storage-blob-encryption-with-cmk/main.bicep:resource keyVault 'Microsoft.KeyVault/vaults@2021-10-01' = {
quickstarts/microsoft.web/function-http-trigger/main.bicep:resource keyVault 'Microsoft.KeyVault/vaults@2019-09-01' = {
quickstarts/microsoft.web/function-http-trigger/main.bicep:resource keyVaultSecret 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = {
quickstarts/microsoft.web/private-webapp-with-app-gateway-and-apim/main.bicep:resource keyVaultPrivateEndpoint 'Microsoft.Network/privateEndpoints@2021-02-01' = {
quickstarts/microsoft.web/private-webapp-with-app-gateway-and-apim/main.bicep:  resource keyVaultPrivateDnsZoneGroup 'privateDnsZoneGroups' = {
quickstarts/microsoft.web/private-webapp-with-app-gateway-and-apim/main.bicep:resource keyVaultPrivateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
quickstarts/microsoft.web/private-webapp-with-app-gateway-and-apim/main.bicep:  resource keyVaultPrivateDnsZoneLink 'virtualNetworkLinks' = {
quickstarts/microsoft.web/private-webapp-with-app-gateway-and-apim/main.bicep:resource keyVault 'Microsoft.KeyVault/vaults@2021-04-01-preview' = {
```

This one shows us the locations in the repository -- that is the file names. How do we find the history of modifications to it? Turns out, `git log` can search by line reference:


```bash
git log -L:"resource keyVault:quickstarts/microsoft.web/private-webapp-with-app-gateway-and-apim/main.bicep" --no-patch --oneline
ef64cb155 New quickstart showing Application Gateway with internal API Management and Web App (#11939)
```
* here, I am taking the last line from the previos command output

What if along with the commit message, we wanted to see the author? For such cases `git log` supports formatting:

```bash
$ git log -L:"resource keyVault:quickstarts/microsoft.web/private-webapp-with-app-gateway-and-apim/main.bicep" --no-patch --oneline --pretty="%h %s || %an %ae" | tail -1
ef64cb155 New quickstart showing Application Gateway with internal API Management and Web App (#11939) || Michael S. Collier mcollier@microsoft.com
```