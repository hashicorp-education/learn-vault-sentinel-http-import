# Restrict enterprise namespace group members with Sentinel

This lab illustrates how you can use Sentinel and its [http](https://docs.hashicorp.com/sentinel/imports/http/) import module in Vault Enterprise to restrict the subgroups and entities of a Vault group.

It requires all subgroups and entities of a group in Vault to belong to the same namespace as that group or to one of its descendant namespaces. This also includes children, grandchildren, great-grandchildren, etc.

This is also a nice example of how the `http` import makes Sentinel in Vault much more powerful.

> **NOTE:** The policies in this lab require Vault Enterprise version 1.5 or higher since the `http` import was not included in earlier versions of Vault Enterprise.

Before starting this lab, you might want to learn about the [Identity Secrets Engine](https://www.vaultproject.io/docs/secrets/identity) and [Namespaces](https://www.vaultproject.io/docs/enterprise/namespaces).

 You should also learn about [Sentinel](https://docs.hashicorp.com/sentinel/) and how it works in Vault in this [document](https://www.vaultproject.io/docs/enterprise/sentinel) if you are not already familiar.

## Policies

The repository directory holds 2 Sentinel [Endpoint Governing Policies](https://www.vaultproject.io/docs/enterprise/sentinel#endpoint-governing-policies-egps) (EGPs):

1. [get-namespace-map.sentinel](./get-namespace-map.sentinel)

1. [restrict-namespaces-of-group-members.sentinel](./restrict-namespaces-of-group-members.sentinel)

Create both of these policies in the root namespace of your Vault cluster using the Vault UI, CLI, or HTTP API.

## The get-namespace-map policy

The first policy, `get-namespace-map.sentinel`, builds a complete map of all namespaces in a Vault cluster. It returns a JSON map with the names of all Vault namespaces as keys and lists of descendant namespaces of the keys' namespaces as values. You can periodically invoke this policy to retrieve the current namespace map.

The first policy uses Sentinel's [http](https://docs.hashicorp.com/sentinel/imports/http/) import make calls against the [/sys/namespaces](https://www.vaultproject.io/api-docs/system/namespaces#list-namespaces) endpoint of Vault's HTTP API. The policy handles this in the `get_namespace_map` function which is initially called against the root namespace ("") but calls itself recursively against each discovered child namespace until it has explored all namespaces.

You will apply the first policy against the path, `secret/get-namespace-map` in the root namespace of your Vault cluster. Since the policy always returns `false`, you can invoke it and get the current namespace map like this:

```shell
vault read secret/get-namespace-map
```

You will write this output with the `vault write` command to the static secret, `secret/current-namespace-map`. You learn how to do this in detail in the **Namespaces** section.

## The restrict-namespaces-of-group-members policy

The second policy, `restrict-namespaces-of-group-members.sentinel`, reads the namespace map generated by the first policy from the path `secret/current-namespace-map`. It then checks all subgroups listed in the `member_group_ids` attribute and all entities listed in the `member_entity_ids` attribute of the group pending creation. It verifies whether the namespace that subgroup or entity belongs to is in the list of namespaces associated with the parent group's namespace. 

The end result is that all subgroups and entities of a group must belong to the same namespace as that group or to one of its descendant namespaces.

You create the second policy in the root namespace. It's applied to all paths in all namespaces that match `identity/group(.*)`. This occurs by actually applying the policy to the path `*` in the root namespace but then using Sentinel's [when](https://docs.hashicorp.com/sentinel/language/rules/#when-predicates) predicate to do any significant processing against paths that match `identity/group(.*)`.

The `main` rule handles this:

```
main = rule when request.path matches "identity/group(.*)" and
								 request.operation in ["create", "update"] {
  validate_group(request.data, namespace.path)
}
```

Like the first policy, the second policy uses Sentinel's `http` import to make calls against the Vault HTTP API:

1. It first calls the `secret/current-namespace-map` endpoint to fetch the namespace map.

1. Then it makes calls against the `identity/group/id/<subgroup>` API endpoint for each subgroup of the group requiring creation. It actually does this against all namespaces until finding the group ID (which occurs when you get a `200` response code), because there is no way to query a group without knowing its namespace. You're essentially just calling the Vault API endpoint to find the namespace the subgroup belongs to.

1. At that point, you can check whether the discovered namespace of the subgroup is in the list of allowed namespaces associated with the group's namespace in the namespace map; these are the descendant namespaces and the group's own namespace.

1. Then the policy makes calls against the `identity/entity/id/<entity>` API endpoint for each entity of the group requiring creation. Again, calls to this API endpoint happen against all namespaces until you get a `200` response code which indicates that you have found the namespace that has the entity.

1. At that point, you can check whether the discovered namespace of the entity is in the list of allowed namespaces associated with the group's namespace in the namespace map; these are the descendant namespaces and the group's own namespace.

If any subgroups or entities assigned to the group you request to create are not in the group's own namespace or that namespace's descendants, the policy fails and group creation fails.

## Using the policies

To use these policies, you will need to run a Vault Enterprise server or cluster with some namespaces defined. You should also create some groups and entities in different namespaces.

You must replace `<your_vault_domain>` in the `vault_addr` parameter of both policies with the domain of your Vault Enterprise server or cluster. You must also replace `<your_token>` in the `vault_token` parameter of both Sentinel policies with an actual Vault token in the root namespace of that server or cluster. The token needs all the following Vault ACL policy capabilities:

- Write to and read from `secret/get-namespace-map` and `secret/current-namespace-map`.

- List and read from `sys/namespaces` in all namespaces.
	
- Read from `identity/group/id/*` in all namespaces.
	
- Read from `identity/entity/id/*` in all namespaces.

Here is a partial example of what such a policy in the root namespace might look like, but you would need to customize to adjust to your namespace map:

```plaintext
# Read from and write to secret/get-namespace-map
path "secret/get-namespace-map" {
   capabilities = ["create", "read", "update"]
}

# Read from and write to secret/current-namespace-map
path "secret/current-namespace-map" {
   capabilities = ["create", "read", "update"]
}

# Read and list namespaces in all namespaces
path "sys/namespaces/*" {
   capabilities = ["read", "list"]
}
path "Sales/sys/namespaces/*" {
   capabilities = ["read", "list"]
}
path "Sales/SE/sys/namespaces/*" {
   capabilities = ["read", "list"]
}
path "Sales/CAM/sys/namespaces/*" {
   capabilities = ["read", "list"]
}
path "CustomerSuccess/sys/namespaces/*" {
   capabilities = ["read", "list"]
}

# Read groups in all namespaces
path "identity/group/id/*" {
  capabilities = ["read"]
}
path "Sales/identity/group/id/*" {
  capabilities = ["read"]
}
path "Sales/SE/identity/group/id/*" {
  capabilities = ["read"]
}
path "Sales/CAM/identity/group/id/*" {
  capabilities = ["read"]
}
path "CustomerSuccess/identity/group/id/*" {
  capabilities = ["read"]
}

# Read entities in all namespaces
path "identity/entity/id/*" {
  capabilities = ["read"]
}
path "Sales/identity/entity/id/*" {
  capabilities = ["read"]
}
path "Sales/SE/identity/entity/id/*" {
  capabilities = ["read"]
}
path "Sales/CAM/identity/entity/id/*" {
  capabilities = ["read"]
}
path "CustomerSuccess/identity/entity/id/*" {
  capabilities = ["read"]
}
```

### Namespaces

This lab scenario uses nested namespaces several layers deep.

The hierarchy is as follows:

```plaintext
Root
   ├── CustomerSuccess
   │   └── Implementation
   └── Sales
       ├── CAM
       │   ├── East
       │   └── West
       └── SE
           ├── cloudy
           ├── east-SEs
           ├── midwest-SEs
           │   └── kcorbin
           └── west-SEs
```

Read the secret, triggering the first policy:

```shell
vault read secret/get-namespace-map
```

This gives the following output:

```plaintext
Error reading secret/get-namespace-map: Error making API request.

URL: GET https://cam-vault.hashidemos.io:8200/v1/secret/get-namespace-map
Code: 403. Errors:

* 2 errors occurred:
	* egp standard policy "root/get-namespace-map" evaluation resulted in denial.

The specific error was:
<nil>

A trace of the execution for policy "root/get-namespace-map" is available:

Result: false

Description: function that builds map of all namespaces with each
one associated with namespace paths of itself and all
children namespaces

print() output:

Namespace path:
Getting namespaces in the root namespace
Found namespace: CustomerSuccess/
Found namespace: CustomerSuccess/Implementation/
Found namespace: Sales/
Found namespace: Sales/SE/
Found namespace: Sales/SE/midwest-SEs/
Found namespace: Sales/SE/midwest-SEs/kcorbin/
Found namespace: Sales/SE/west-SEs/
Found namespace: Sales/SE/cloudy/
Found namespace: Sales/SE/east-SEs/
Found namespace: Sales/CAM/
Found namespace: Sales/CAM/West/
Found namespace: Sales/CAM/East/
```

followed by the namespace map, with added line breaks below to make it easier to read:

```plaintext
Complete Namespace Map:
{
  "":["","CustomerSuccess/","CustomerSuccess/Implementation/","Sales/","Sales/CAM/","Sales/CAM/West/","Sales/CAM/East/","Sales/SE/","Sales/SE/cloudy/","Sales/SE/east-SEs/","Sales/SE/midwest-SEs/","Sales/SE/midwest-SEs/kcorbin/","Sales/SE/west-SEs/"],
  "CustomerSuccess/":["CustomerSuccess/","CustomerSuccess/Implementation/"],
  "CustomerSuccess/Implementation/":["CustomerSuccess/Implementation/"],
  "Sales/":["Sales/","Sales/CAM/","Sales/CAM/West/","Sales/CAM/East/","Sales/SE/","Sales/SE/cloudy/","Sales/SE/east-SEs/","Sales/SE/midwest-SEs/","Sales/SE/midwest-SEs/kcorbin/","Sales/SE/west-SEs/"],
  "Sales/CAM/":["Sales/CAM/","Sales/CAM/West/","Sales/CAM/East/"],
  "Sales/CAM/East/":["Sales/CAM/East/"],
  "Sales/CAM/West/":["Sales/CAM/West/"],
  "Sales/SE/":["Sales/SE/","Sales/SE/cloudy/","Sales/SE/east-SEs/","Sales/SE/midwest-SEs/","Sales/SE/midwest-SEs/kcorbin/","Sales/SE/west-SEs/"],
  "Sales/SE/cloudy/":["Sales/SE/cloudy/"],
  "Sales/SE/east-SEs/":["Sales/SE/east-SEs/"],
  "Sales/SE/midwest-SEs/":["Sales/SE/midwest-SEs/","Sales/SE/midwest-SEs/kcorbin/"],
  "Sales/SE/midwest-SEs/kcorbin/":["Sales/SE/midwest-SEs/kcorbin/"],
  "Sales/SE/west-SEs/":["Sales/SE/west-SEs/"]
}
```

> **NOTE:** You do not need to add any line breaks to the generated namespace map before writing it to the `secret/current-namespace-map` secret. 

You can do that like this:

```shell
vault write secret/current-namespace-map namespace-map='<generated_namespace_map>'
```

where `<generated_namespace_map>` is the json returned after "Complete Namespace Map".

If this works, you should observe 

```plaintext
"Success! Data written to: secret/current-namespace-map".
```

### Groups and entities

You have the following group IDs that you tried to add when creating a new group in the Sales namespace:

```
0eca3132-c0ec-5807-9014-549fc91fc99e in Sales/SE
c81732b0-cea3-3a6c-0a52-d3b4afaca440 in Sales/SE
e9e2a598-0a93-1759-8851-b1a191e6dd9c in CustomerSuccess
```

You have the following entity IDs that you tried to add when creating a new group in the Sales namespace:

```
870a6d92-10f0-55b2-805d-5a3411a8bdd4 in Sales
93c0685e-5082-2f10-96c8-d0d42c9b1527 in Sales/SE/east-SEs
96b1876e-dbc7-187a-956e-e4ff035798cd in the root namespace
```

The `restrict-namespaces-of-group-members` policy allows the first two groups and the first two entities, while it doesn't allow the last group and the last entity.

## What the policy gives you

Test the `restrict-namespaces-of-group-members` policy with the following commands. Example results returned by Sentinel appear after each command.

The first command tries to create an invalid group since the third subgroup and the third entity are in invalid namespaces. This triggers a Sentinel policy violation, so Vault cannot create the group.

The second command leaves out the third subgroup and the third entity. 

Vault creates the group.

### Create an invalid group

The following command triggers a Sentinel policy violation. The group is not created.

```shell
vault write Sales/identity/group name=test-sales-2 policies="default" member_group_ids="c81732b0-cea3-3a6c-0a52-d3b4afaca440","0eca3132-c0ec-5807-9014-549fc91fc99e","e9e2a598-0a93-1759-8851-b1a191e6dd9c" member_entity_ids="870a6d92-10f0-55b2-805d-5a3411a8bdd4","93c0685e-5082-2f10-96c8-d0d42c9b1527","96b1876e-dbc7-187a-956e-e4ff035798cd"
```

This gives:

```plaintext
Error writing data to Sales/identity/group: Error making API request.

URL: PUT https://cam-vault.hashidemos.io:8200/v1/Sales/identity/group
Code: 403. Errors:

* 2 errors occurred:
	* egp standard policy "root/validate-group-2" evaluation resulted in denial.

The specific error was:
<nil>

A trace of the execution for policy "root/validate-group-2" is available:

Result: false

Description: Validate that a group only contain sub-groups from the
same or children namespaces using namespace paths

print() output:

Namespace path: Sales/
Request path: identity/group
Request data: {"member_entity_ids": "870a6d92-10f0-55b2-805d-5a3411a8bdd4,93c0685e-5082-2f10-96c8-d0d42c9b1527,96b1876e-dbc7-187a-956e-e4ff035798cd", "member_group_ids": "c81732b0-cea3-3a6c-0a52-d3b4afaca440,0eca3132-c0ec-5807-9014-549fc91fc99e,e9e2a598-0a93-1759-8851-b1a191e6dd9c", "name": "test-sales-2", "policies": "default"}
Request operation: update
namespace_map: {"": ["" "CustomerSuccess/" "CustomerSuccess/Implementation/" "Sales/" "Sales/CAM/" "Sales/CAM/West/" "Sales/CAM/East/" "Sales/SE/" "Sales/SE/cloudy/" "Sales/SE/east-SEs/" "Sales/SE/midwest-SEs/" "Sales/SE/midwest-SEs/kcorbin/" "Sales/SE/west-SEs/"], "CustomerSuccess/": ["CustomerSuccess/" "CustomerSuccess/Implementation/"], "CustomerSuccess/Implementation/": ["CustomerSuccess/Implementation/"], "Sales/": ["Sales/" "Sales/CAM/" "Sales/CAM/West/" "Sales/CAM/East/" "Sales/SE/" "Sales/SE/cloudy/" "Sales/SE/east-SEs/" "Sales/SE/midwest-SEs/" "Sales/SE/midwest-SEs/kcorbin/" "Sales/SE/west-SEs/"], "Sales/CAM/": ["Sales/CAM/" "Sales/CAM/West/" "Sales/CAM/East/"], "Sales/CAM/East/": ["Sales/CAM/East/"], "Sales/CAM/West/": ["Sales/CAM/West/"], "Sales/SE/": ["Sales/SE/" "Sales/SE/cloudy/" "Sales/SE/east-SEs/" "Sales/SE/midwest-SEs/" "Sales/SE/midwest-SEs/kcorbin/" "Sales/SE/west-SEs/"], "Sales/SE/cloudy/": ["Sales/SE/cloudy/"], "Sales/SE/east-SEs/": ["Sales/SE/east-SEs/"], "Sales/SE/midwest-SEs/": ["Sales/SE/midwest-SEs/" "Sales/SE/midwest-SEs/kcorbin/"], "Sales/SE/midwest-SEs/kcorbin/": ["Sales/SE/midwest-SEs/kcorbin/"], "Sales/SE/west-SEs/": ["Sales/SE/west-SEs/"]}

allowed_namespaces: ["Sales/" "Sales/CAM/" "Sales/CAM/West/" "Sales/CAM/East/" "Sales/SE/" "Sales/SE/cloudy/" "Sales/SE/east-SEs/" "Sales/SE/midwest-SEs/" "Sales/SE/midwest-SEs/kcorbin/" "Sales/SE/west-SEs/"]
Evaluating member_group_ids
Group c81732b0-cea3-3a6c-0a52-d3b4afaca440 in namespace Sales/SE/ is allowed
Group 0eca3132-c0ec-5807-9014-549fc91fc99e in namespace Sales/SE/ is allowed
Group e9e2a598-0a93-1759-8851-b1a191e6dd9c in namespace CustomerSuccess/ is not allowed
Evaluating member_entity_ids
Entity 870a6d92-10f0-55b2-805d-5a3411a8bdd4 in namespace Sales/ is allowed
Entity 93c0685e-5082-2f10-96c8-d0d42c9b1527 in namespace Sales/SE/east-SEs/ is allowed
Entity 96b1876e-dbc7-187a-956e-e4ff035798cd in namespace  is not allowed


Rule "main" (byte offset 4181) = false
  true (offset 977): "data" in keys(body_nsm)
  true (offset 1006): "namespace_map" in keys(body_nsm.data)

	* permission denied
```

### Create an allowed group

The following command does not trigger a Sentinel policy violation, and Vault creates the group.

```shell
vault write Sales/identity/group name=test-sales-2 policies="default" member_group_ids="c81732b0-cea3-3a6c-0a52-d3b4afaca440","0eca3132-c0ec-5807-9014-549fc91fc99e" member_entity_ids="870a6d92-10f0-55b2-805d-5a3411a8bdd4","93c0685e-5082-2f10-96c8-d0d42c9b1527"
```

This returns:

```plaintext
Key     Value
---     -----
id      bcacb6d4-3b83-3c62-73e2-88b3e5fea0af
name    test-sales-2
```

## Conclusion

You used this lab to show that the Sentinel `http` import allows Sentinel policies in Vault Enterprise 1.5 or higher to make calls against the Vault HTTP API. This makes Sentinel in Vault much more powerful.

You tried 2 policies that used the `http` import to make calls against three of Vault's HTTP API endpoints related to namespaces, groups, and entities. The first policy generated a map of all namespaces in the Vault cluster. The second policy limited the creation of groups, requiring that all the group's subgroups and entities belong to the group's namespace or to descendant namespaces of it.
