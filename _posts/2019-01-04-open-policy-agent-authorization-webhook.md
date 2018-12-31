---
title: "Kubernetes Authorization via Open Policy Agent"
categories:
  - Kubernetes
tags:
  - Kubernetes
  - OpenPolicyAgent
---

# Introduction

In a best-practice Kubernetes cluster every request to the Kubernetes APIServer is authenticated and authorized. [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) is usually implemented by the [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) authorization module. But there are alternatives, this blog post explains how to implement advanced authorization policies via [Open Policy Agent (OPA)](https://www.openpolicyagent.org/) by leveraging the [Webhook](https://kubernetes.io/docs/reference/access-authn-authz/webhook/) authorization module.


# Motivation

We are a team providing managed Kubernetes clusters to our company-internal customers. To provide a near upstream Kubernetes experience, we want to grant our customers cluster-admin-like access. But to ensure baseline security and stability, we don't want to grant full cluster-admin privileges. For example:
 * We want to allow full access to any namespace except `kube-system`, because our infrastructure (e.g. monitoring & logging) is deployed there. 
 * We want to enforce a PodSecurityPolicy which doesn't allow running containers as `root` user.
  
 Our first implementation was implemented via Kubernetes RBAC and a custom operator. The basic idea was to grant all necessary rights via RBAC RoleBindings. So we gave our customers the ClusterRole `admin` for every namespace except `kube-system` (via the operator). Every time we found that something wasn't working as expected we added additional rights, either via a per-namespace Role or via a ClusterRole. This lead to a lot of individual rules for specific use-cases and wasn't really maintainable in the long term. Especially as our user base continues to grow it's not really feasible to adjust the Roles whenever somebody detects an edge cases which doesn't work with our configuration.   
 
 So instead of configuring authorization based on a whitelist we switched to a blacklist-based model. What we actually wanted was to give our customers cluster-admin access and only restrict some specific rights. Therefore an implementation based on a blacklist via Open Policy Agent was a natural fit.

## Whitelist vs. Blacklist

Most requirements regarding Authorization can be implemented by simply using the RBAC authorization module via Roles and RoleBindings, which are explained in [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). But RBAC is by design limited to whitelisting, i.e. for every requests it's checked if one of the Roles and RoleBindings apply and in that case the request is approved. Requests are only denied if there is no match, there is no way to deny requests explicitly. At first this doesn't sound like a big limitation, but some specific use cases require more flexibility. For example:
* A user should be able to create/ update/ delete pods in all namespaces except `kube-system`. The only way to implement this via RBAC is to assign the rights on a per-namespaces basis, e.g. by deploying a ClusterRole and a per-namespace RoleBinding. If the namespaces change over time you have to either deploy this RoleBindings manually or run an operator for this.
* A Kubernetes cluster is provided with pre-installed StorageClasses. A user should be able to create/ update/ delete custom StorageClasses, but he shouldn't be able to modify the pre-installed ones. If this would be implemented via RBAC, the user must have the right to create StorageClasses and as soon as he creates a StorageClass additional rights must be assigned to update and delete this StorageClass. As above, this could be implemented via an operator.  
When you have lot of this use cases, you'll get a lot of custom logic implemented via operators. Sooner or later this doesn't scale, because with a lof of operators and accompanying RBAC Roles it gets really hard to understand what rights a user actually has. We will show that both cases can be implemented easier via Open Policy Agent.

## Webhook Authorization Module vs. ValidatingWebhook & MutatingWebhook

Some advanced use cases can also be implemented via [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/), i.e. ValidatingWebhook or MutatingWebhook. There are also blog posts which dive into how Open Policy Agent can be used for this: [Policy Enabled Kubernetes with Open Policy Agent](https://medium.com/@jimmy.ray/policy-enabled-kubernetes-with-open-policy-agent-3b612b3f0203) and [Kubernetes Compliance with Open Policy Agent](https://itnext.io/kubernetes-compliance-with-open-policy-agent-3d282179b1e9). Dynamic Admission Control has the limitation that the webhooks are only called for create, update and delete events on Kubernetes resources. So it's for example impossible to deny get requests. But they also have advantages compared to the Webhook authorization module because they can deny requests based on the content of a Kubernetes resource. That's information the Webhook authorization module has no access to. For reference, the Webhook authorization module decides based on [SubjectAccessReviews](https://github.com/kubernetes/kubernetes/blob/master/pkg/apis/authorization/types.go#L30), whereas the ValidatingWebhook and MutatingWebhook decide based on [AdmissionReviews](https://github.com/kubernetes/api/blob/master/admission/v1beta1/types.go#L29). In our implementation we've integrated OPA via authorization module and via MutatingWebhook. 
 

# Overview

This section shows on a conceptual level how Kubernetes is integrated with Open Policy Agent. Because the Open Policy Agent itself doesn't implement the REST interface required by Kubernetes, the Kubernetes Policy Controller translates Kubernetes SubjectAccessReviews and AdmissionReviews into Open Policy Agent queries.

<!-- TODO: Change to ![overview](https://raw.githubusercontent.com/sbueringer/sbueringer.github.io/master/images/2019-01-04-open-policy-agent-authorization-webhook.png) -->
![Overview](https://raw.githubusercontent.com/sbueringer/sbueringer.github.io/opa-post/images/2019-01-04-open-policy-agent-authorization-webhook.png)

For every request the Kubernetes API Server receives, the following sequence is executed:
1. The request is authenticated.
1. Based on the user information extracted by the authentication the request is authorized:
    1. First the Webhook is called. In our case, the Webhook can either deny the request or forward it to RBAC. The Kubernetes Webhook can also allow requests, but that's not implemented in Kubernetes Policy Controller.
    1. Second the RBAC module is executed. If RBAC doesn't allow the request, the request is denied.
1. If the request leads to a change in the persistence, e.g. create/ update/ delete a resource, Admission Controllers are executed (MutatingWebhook is only one of them). Read more about Admission Controllers [here](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/). 

So depending on what exactly we want to deny, we now can implement authorization or admission OPA policies. Further information how this scenario is configured can be found here [Azure/kubernetes-policy-controller Authorization scenario](https://github.com/Azure/kubernetes-policy-controller).


# Examples

This section shows how this setup is used to implement the use cases described above:

## Create/ update/ delete pods in every namespace except kube-system

First of all, we grant the group `user` the rights to create/ update/ delete pods via RBAC:

````yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pods
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "update", "delete"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: user-pods
subjects:
- kind: Group
  name: user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pods
  apiGroup: rbac.authorization.k8s.io
````

Now every user in the group `user` is allowed to create/ update/ delete pods cluster-wide. To restrict these rights via OPA the following policy is deployed:

````
package authorization

import data.k8s.matches

deny[{
	"id": "pods-kube-system",
	"resource": {
		"kind": kind,
		"namespace": namespace,
		"name": name,
	},
	"resolution": {"message": "Your're not allowed to create/ update/ delete pods in kube-system"},
}] {
	matches[[kind, namespace, name, resource]]

	not re_match("^(system:kube-controller-manager|system:kube-scheduler)$", resource.spec.user)
	resource.spec.resourceAttributes.namespace = "kube-system"
	resource.spec.resourceAttributes.resource = "pods"
	re_match("^(create|update|delete|deletecollections)$", resource.spec.resourceAttributes.verb)
}
```` 

Notes:
* We have to exclude `system:kube-controller-manager` and `system:kube-scheduler`, because both the Kubernetes controller manager as well as the scheduler have to be able to access pods.
* By simply removing `resource.spec.resourceAttributes.resource = "pods"` from the policy, we could restrict access to all namespaced resources in `kube-system`. 
* We have to be careful to deny the correct verbs. A simple `delete` in RBAC allows `delete` and  `deletecollection` (cf. [Authorization Overview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)).
* Open Policy Agent makes it also very easy to write unit tests for all our policies. For more information, see [How Do I Test Policies?](https://www.openpolicyagent.org/docs/how-do-i-test-policies.html)

## Create/ update/ delete on specific StorageClasses 

As in the first example, we first have to grant user access via RBAC:

````yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: storageclasses
rules:
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["create", "update", "delete"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: user-storageclasses
subjects:
- kind: Group
  name: user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: storageclasses
  apiGroup: rbac.authorization.k8s.io
````

In our case, we only want to restrict access to a StorageClass called `ceph`. So we're deploying the following policy:

````
package authorization

import data.k8s.matches

deny[{
	"id": "storageclasses",
	"resource": {
		"kind": kind,
		"namespace": namespace,
		"name": name,
	},
	"resolution": {"message": "Your're not allowed to create/update/delete the StorageClass 'ceph'"},
}] {
	matches[[kind, namespace, name, resource]]

	resource.spec.resourceAttributes.resource = "storageclasses"
	resource.spec.resourceAttributes.name = "ceph"
	re_match("^(create|update|delete|deletecollections)$", resource.spec.resourceAttributes.verb)
}
````

An unit test for this policy can be implemented like this:

````

test_deny_update_storageclass_ceph {
	deny[{"id": id, "resource": {"kind": "storageclasses", "namespace": "", "name": "ceph"}, "resolution": resolution}] with data.kubernetes.storageclasses[""].ceph as {
		"kind": "SubjectAccessReview",
		"apiVersion": "authorization.k8s.io/v1beta1",
		"spec": {
			"resourceAttributes": {
				"verb": "update",
				"version": "v1",
				"resource": "storageclasses",
				"name": "ceph",
			},
			"user": "alice",
			"group": ["user"],
		},
	}
}
````

# Conclusion

In summary, OPA allows far more flexible policies compared to the built-in RBAC authorization, especially if no additional operator is used. In my opinion it would be nice to have direct integration for OPA as Authorization Module and Admission Controller, but in the meantime Kubernetes Policy Controller bridges the gap between Kubernetes and OPA. Some inspirations what can be implemented:
* Deny access to specific CustomResourceDefinitions, e.g. `calico`
* Deny access to specific ClusterRoles, e.g. `cluster-admin`, `admin`, `edit`, `view`
* Allow port-forward only to some specific pods in `kube-system`
* Create a mapping which PodSecurityPolicies can be used in which namespaces
* Allow access to ValidatingWebhookConfigurations except some pre-installed ones

What do you think about Open Policy Agent as policy engine for Kubernetes? What are your use-cases and are they already covered by RBAC? If not, what would you like to implement via the Open Policy Agent?

Thanks to all contributors of Open Policy Agent and Kubernetes Policy Controller Special thanks to [Nikhil Bhatia](https://github.com/rite2nikhil) and [Torin Sandall](https://github.com/tsandall) for their support for our implementation of the authorization module integration of Open Policy Agent.