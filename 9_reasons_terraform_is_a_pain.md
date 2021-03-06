# 9 reasons why Terraform is a pain, and 1 why you should still care.

Working with Terraform can be difficult and cumbersome, but its still worth it.

## Background story

Back in 2015, when I first found out about Terraform, it looked like a Valhalla to me. 
Terraform was about to solve the issue of provisioning complicated infrastructure - 
bringing together worlds of multiple cloud providers - ranging from multi-purpose giants 
like AWS to one-solution providers like Logentries.

Together with my team we decided, that we need something to deal with the infrastructure complexity
we have. For a platform based on Heroku and AWS, scaled horizontally to four clones
Terraform seemed like a perfect solution. We wanted to have something that would let us
realize the idea of Insfrastructre as a Code - a must for a 
[DevOps enabled](https://smarketshq.com/devops-is-a-culture-not-a-title-prefix-b825b44dfcda) team.
Advanced and feature-full Terraform is, it doesn't come free - there is a couple of issues that 
you should be aware of. 

I will enumerate the ones that hurted us the most, and show you our means to deal with them.
In the end I will try to convince You that, even with those challenges, there is a lot 
of room for Terraform in the tooling space.

## The pains

### 1. The evil state

First thing you will complain about, when it comes to Terraform, is the fact that 
it's stateful, and the implications it brings. I personally consider two issues that 
it brings:

- the state has to be in-sync with the infrastructure all the time - that also means that you have to go 
all-in when it comes to provisioning - i.e. no stack modifications can be made outside of the provisioning tool

- you have to keep the state somewhere - and this has to be a secure location as state has to carry secrets

But there is a reason why the state was introduced into Terraform. It's there 
to maintain the mapping between the resources represented in your definition files and 
the actual resources created within cloud providers. 
Having that, Terraform can give you a couple of advantages:
 
- reading the state from providers (state syncing, also called refreshing), can be quite time-consuming. 
 If we could be 100% sure that the state is accurate, we could totally resign from that, and apply the change right away

- having ability to follow the resources that already have been created, we can easier apply renames and 
restructuring modifications - simply an infrastructure refactoring

- when it comes to state, Terraform requires it to be locked before applying the changes. 
That means that we can assure that while we are applying changes no-one else does.

I think when considering provisioning tool you should weigh up above arguments and make sure 
if your stack is more of a clean-sheet kinda thing, that can be recreated every time 
you change something, or is it rather a living organism that requires modifications 
while it's still running.

### 2. Hard to start with the existing stack

Back in the early days of Terraform, its issue tracker was full of complaints from people not being 
able to leverage Terraform with the existing stack. The reason for it was the fact, that Terraform was 
not able to incorporate it into the state (to my amazement, while looking for sign of this, I've found 
[my old PR](https://github.com/hashicorp/terraform/pull/875) that was trying to address that issue back then ;) ). 
Fortunately, the import command was introduced, and this problem has been solved (at least at the system level).

But here comes another issue that is tightly connected to this - if your stack is large you are doomed to issue 
`terraform import` command multiple times for each resource, that is already there. 
Without some nifty automation/scripting, it could be really time consuming and frustrating. 
When you think about it, it would be nice to import such things in a bit more smart way. This however would require 
Terraform to treat resources not as a flatland of resources, but as a tree. In some cases, it makes perfect sense - 
have a look at `heroku_app` vs `heroku_domain` or `heroku_drain.` There is certainly a lot of room for improvement in that space.

### 3. Complicated state modifications

There is one additional thing that is a bit problematic when dealing with the state. While 
constantly refactoring your infrastructure definition, you may end up renaming resources 
(changing their identifiers) or moving them deeper into modules. 
Such changes are unfortunately hard for Terraform to follow, and leave it in a state 
where it doesn't know that certain resources are simply misplaced. 
If you run `apply` again, you will end up in resource recreation, which is probably 
not something You always want. The good news is that there is a `terraform state mv` 
command that allows you to move the logical resource around the state. 
The bad news is that in most of the cases you will need a lot of those.

### 4. Tricky conditional logic

There are some people around the web who doesn't like the fact that Terraform is not really an actual 
imperative programming language. To be perfectly honest I don't share that opinion - I think the provisioning definition 
of the stack should be as declarative as it can - that leaves a lot less space for some 
deviations in the definitions. 
On the other hand, the conditional logic provided by Terraform is a bit tricky. For example to define a resource that 
is conditionally provisioned you make the resource to be a list, and use the count parameter to control it:

```
resource "heroku_app" "some_app" {
    count = "${var.create_app}" 
    name = "some-app"
    ...
}
```
thats rather specific, and you don't really want to know how does if/else look like. Ok, you should:

```
resource "heroku_app" "some_app" {
    count = "${var.create_app}" 
    name = "some-app"
    ...
}

resource "heroku_app" "some_app" {
    count = "${1 - var.create_app}" 
    name = "some-app"
    ...
}
```

So there is a point in saying, that you should stay away from constructs like this as far as you can. Of course, 
it doesn't mean it should be a reason to resign from Terraform because of this, but be warned. There is a nice
[article](https://blog.gruntwork.io/terraform-tips-tricks-loops-if-statements-and-gotchas-f739bbae55f9) from 
Gruntwork about all of the things you can and can't do with count - really worth reading.

In some close release this problem should be simplified with 
[resource for_each](https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each). 
Let's keep our fingers crossed :).

### 5. One can't simply iterate over modules

The actual idea of the module is awesome - it lets you enclose a reusable, set of resources in a reusable artifact. 
Let's have a look at some simplified example:

**app/app.tf**

```
resource "heroku_app" "app" {
  name = "${var.name}"
  region = "eu"
  organization = {
    "name" = "${var.organization}"
  }
  config_vars = ["${var.configuration}"]
}

resource "heroku_addon" "deploy_hook" {
  app = "${heroku_app.app.name}"
  plan = "deployhooks:http"
  config = {
    url = "${var.deployhook_url}"
  }
}

output "name" {
  value = "${heroku_app.app.name}"
}
```

which then can be used in service declaration:

**stack/some_service.tf**

```
module "app" {
  source = "../app"
  name = "some-service"
  configuration = {
    SOME_ENV_VARIABLE = "value"
  }
  pipeline_id = "000000-9207-490a-b050-617b01ef79f3"
  deployhook_url = "https://some.deploy.hook"
}
```

and that was a gamechanger for us, because we had a lot of repeating resources attached to 
each app - monitoring, logdrains, deployhooks (as above) to name a few. 
But there is one really hurting issue that comes with them - for some reason they are 
not representing the same artifact as actual resources. That means, specifically, 
that [they don't support count parameter](https://github.com/hashicorp/terraform/issues/953)
which is critical when applying conditional logic stated above, 
or in our case - iteration over services per each clone. In exact, instead of doing:

**stack/services.tf (this is not real)**

```
module "app" {
    source = "../app"
    name = "some-service-${var.clone[count.index]}"
    configuration = "${var.configuration[count.index]}"
    ...
}
```

we have to repeat the definition per each clone:

**stack/services.tf**

```
module "clone1_app" {
    source = "../app"
    name = "some-service-clone1"
    configuration = "${var.clone1Configuration}"
}

module "clone2_app" {
    source = "../app"
    name = "some-service-clone2"
    configuration = "${var.clone2Configuration}"
}
```
This issue is also promised to be sorted out in a [forseeable future](https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each##module-count-and-for_each).

### 6. Flickering resources

Being a 0.x software and feature-rich piece of software, Terraform carries a huge luggage of tiny 
errors that you might stumble upon. One of those itchy things was the fact that some resources 
don't want to stay in a stable state. For us it was always an SNS topic subscription policy - 
everything we've been doing around service that had a queue subscribed to SNS, it was always 
modifying those (no matter it didn't make much of a sense). It can lead to a lot of confusion - 
esp. when someone touches Terraform for the first time. While this issue is provider-local and 
will be most probably fixed over time, you have all the time have it at the back of your mind.

### 7. Those tiny details

Another tiny issue that we had was the inability to use `count` value that is relying on the 
state of something that is meant to be computed (in modules). Even something like:

```
resource "heroku_app" "sample" {
   count = "${lookup(var.some_map, "enable_sample_app", 0)}" 
   name = "sample-app"
   ...
}
```

when above thing is defined in module, you get a sweet message saying: `value of 'count' cannot be computed`..
It's really annoying - especially when you read the explanation of Hashicorp saying that you can always use 
`-target` switch to [initialize](https://github.com/hashicorp/terraform/issues/12570##issuecomment-318414280)
resources one after another :(.

### 8. How to deal with secrets?

One of the reasons why Terraform files are so hard to be kept around is the question of where to 
keep secrets. There are a couple of ways of dealing with that:

- The Hashicorp's blessed way of doing the thing is to use their Vault - while this could be the way to go, 
it complicates the whole setup even more and feels a little bit like an overkill

- Similar to Vault you can use KMS from AWS to store secrets - but it carries the same 
complexity luggage

- Use a private git repository, and pretend that everything is okay, as long as no one's computer is stolen ;)

- There is also a way of keeping secrets in the env vars. That kinda makes sense when you run
the thing from CD/CI server - though in a sufficiently complicated system, this could be really
hard to maintain

- You could keep them somewhere local, have some special machine that would be exclusively for 
provisioning, but let's face it - for a reasonably sized team that's a 'nogo'.

- The way we dealt with this issue, was to keep all secret.tfvars along the .tf files, but encrypted 
using git-secret. The way it works is that the scripts that are running `terraform plan` and `terraform apply` 
for us first use git secret reveal and do git secret hide right after. While this is not a perfect solution, 
it's at least simple enough to decrease the churn needed to run Terraform from local machines.

### 9. One hosting offering

Initially, neither Hashicorp nor any other company was providing any hosting of Terraform. Being quite a 
complicated piece of software to run (esp. the secrets holding part), there was a niche that had to be fulfilled, 
and finally - it was, by [Terraform Enterprise](https://www.terraform.io/docs/enterprise/index.html). 
Unfortunately, I have no experience with this, so can't tell for sure, how it looks. 
But - assuming it's the same feel stripped from rather problematic issues of dealing with 
state and sensitive data - I hope for the best. What might be considered an issue is the fact that 
using Enterprise mode, leaves our provisioning a bit vendor locked-in.

## So what should I (You) do?

As it's quite visible, Terraform carries some issues that have to be taken into account while choosing 
a provisioning solution. Some of those will eventually be sorted out, others are just 
architectural choices Hashicorp had to make (most probably these were lesser-evil like 
decisions). 
As promised in the title, I should give one major point why should have to consider 
Terraform - in my opinion, there are cases where you simply have no other options. 
It's really hard to find a solution ranging over so many cloud providers. 
Additionally, if your case is a living system, with a lot of infrastructure repetitions 
that undergoes minimal infrastructure changes every day - Terraform is definitely worth 
taking a look.
