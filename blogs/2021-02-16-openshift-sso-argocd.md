---
title: "SSO integration for OpenShift GitOps operator"
date: 2021-02-16T00:00:00-04:00
draft: true
categories:
  - Blog
tags:
  - openshift
  - SSO
  - argocd
  - gitops
---

## Why SSO?


## What are you options for OpenShift AuthN/AuthZ?


## Hands-On: SSO using RHSSO operator for ArgoCD apps

### Pre-requisites
- OpenShift 4.X cluster
- Installation of the following operators
  - Red Hat Single Sign-On (RHSSO) Operator
  - Red Hat OpenShift GitOps 

You can install the RHSSO operator under `keycloak` namespace and can use all other default settings when installing the above operators.

### Steps

Connect to your OpenShift 4.X cluster from command-line so that you can execute `oc` commands.

1. Creating Keycloak CRs

```
oc apply -n keycloak -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/operator-examples/mykeycloak.yaml
```

RHSSO Operator uses KeycloakRealm Custom Resources to create and manage Realm resources. Next, create a Keycloak Realm (not using the default `master` realm):

```
oc apply -n keycloak -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/operator-examples/myrealm.yaml
```

The following command will create a new User within Keycloak Realm matched by realmSelector. The newly created User will have username set to "myuser" and password set to "12345":
```
oc -n keycloak apply -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/operator-examples/myuser.yaml
```
2. Access the Keycloak admin console

Before logging into the Keycloak Admin Console, you need to check what is the Admin Username and Password:

```
oc -n keycloak get secret credential-mykeycloak -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```

Now, run the following block of code to find out important Keycloak URLs:

```
KEYCLOAK_URL=https://$(oc -n keycloak get route keycloak --template='{{ .spec.host }}')/auth &&
echo "" &&
echo "Keycloak:                 $KEYCLOAK_URL" &&
echo "Keycloak Admin Console:   $KEYCLOAK_URL/admin" &&
echo "Keycloak Account Console: $KEYCLOAK_URL/realms/myrealm/account" &&
echo ""
```
Navigate to the Keycloak Admin console and make sure that you can log in using the admin credentials from the previous step.

3. Accessing the out-of-the-box ArgoCD instance

On installation of the OpenShift GitOps Operator, the operator sets up an out-of-the-box (OOTB) ArgoCD for cluster configuration.   You can launch into this ArgoCD instance from Console Application Launcher (screenshot below) or from the topology view.

![Default ArgoCD Launcher](../assets/images/rhsso-argocd/default-argocd-instance-launcher.png)

For the pre-created ArgoCD instance under openshift-gitops project, find the password by following these steps:

- Switch to the developer perspective
- Navigate to the “openshift-gitops” project
- Go to “Secrets” tab and find the secret `<argocd-instance-name>-cluster` e.g. “argocd-cluster-cluster” in this case

![Accessing the secret - step1](../assets/images/rhsso-argocd/access-secret-1.png)

![Accesscing the secret - step2](../assets/images/rhsso-argocd/access-secret-2.png)

- Copy the secret and log in to the OOTB ArgoCD server using `admin` and that secret

Alternatively, you may fetch the same using the command line (be sure to update the name of the secret if using a different ArgoCD instance):

```
oc get secret argocd-cluster-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

At this point, you'll still see the default login screen with no option for SSO.

![default login with no SSO](../assets/images/rhsso-argocd/argocd-default-login.png)

1. Creating a new client in Keycloak

First we need to set up a new Keycloak client. Start by logging into your Keycloak server, select the realm you want to use (myrealm is the one we just created) and then go to **Clients** and click the **create** button top right.

![creating a new Keycloak client](../assets/images/rhsso-argocd/new-client.png)

![adding ArgoCD client](../assets/images/rhsso-argocd/adding-argocd-client.png)

Be sure to change the root URL to your ArgoCD server URL. Once you click `Save`, configure the client according to the following:

![configuring ArgoCD client](../assets/images/rhsso-argocd/argocd-client-configuration.png)

If you've filled-out the `Root URL` before, some of the fields would be pre-populated. The important fields to note are the `Access Type` which is set to `confidential` and `Base URL` which is set to `/applications`.

Make sure to click `Save`. You should now have a new tab called `Credentials`. You can copy the `Secret` that we'll use in our ArgoCD configuration later.

![ArgoCD client secret](../assets/images/rhsso-argocd/argocd-client-secret.png)

5. Configuring the groups claim

In order for ArgoCD to provide the groups the user is in we need to configure a groups claim that can be included in the authentication token. To do this we'll start by creating a new `Client Scope` called `groups`.

![add client scopes](../assets/images/rhsso-argocd/add-client-scopes.png)

Once you've created the client scope you can now add a Token Mapper which will add the groups claim to the token when the client requests the groups scope. Make sure to set the Name as well as the Token Claim Name to `groups` and the Mapper Type as `Group Membership`.

![create protocol mapper](../assets/images/rhsso-argocd/create-protocol-mapper.png)

We can now configure the client to provide the `groups` scope. You can now assign the groups scope either to the `Assigned Default Client Scopes` or to the `Assigned Optional Client Scopes`. If you put it in the Optional category you will need to make sure that ArgoCD requests the scope in it's OIDC configuration. Let's use `Assigned Default Client Scopes` by selecting `groups` from the `Available Client Scopes` and clicking `Add selected` option.

![assign default client scope](../assets/images/rhsso-argocd/add-default-client-scopes.png)



