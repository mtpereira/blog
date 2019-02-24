---
title: "Using ejson to store secrets for Terraform"
aliases:
 - "terraform-ejson.html"
 - "terraform-ejson"
date: 2016-11-06T10:21:42Z
tags: ["terraform", "ejson", "secrets"]
draft: false
---

I've been using [Terraform](https://terraform.io) to provision and configure
some application deployed on Heroku, as well as some other infrastructure pieces
on AWS. Since these configurations are kept on a git repository, I do not want any
configuration values stored in clear text, to avoid having any sensitive
information leaked. For this reason, I've searched for a solution to keep these
secrets stored in a safe manner without too much configuration overhead. I've
decided to give [EJSON](https://github.com/Shopify/ejson) a try for this
particular use case, for a couple of reasons:

- It performs asymmetric encryption on the secrets with a pair of keys that are
easy to share with your team, since the public key is actually stored on the
`.ejson` file and the private key is just a string;
- It's installation is pretty simple, since EJSON is a Go binary and it's
  actually available on rubygems, so you can `gem install ejson`;
- It does not require any other piece of infrastructure besides your
  workstation;
- The secrets are kept in JSON files, with the keys in clear text and the values
  encrypted.

This last bit is actually more interesting than it might look, since Terraform's
`.tfvars` files, which store the configurations' variables, are either JSON or
HCL files. As such, we can simply decrypt our EJSON file without any
other transformation or adaptation and pass it directly to Terraform!

With all that being said, let's go through how to achieve this setup and how to use it
efficiently.

# Setup

Assuming you're on macOS, here's how to install all the
required tools:

{{< highlight shell >}}
brew install terraform
gem install ejson
{{</ highlight >}}

Now we need to generate a keypair for EJSON. By default, these are stored at
`/opt/ejson/keys`. To keep it simple and to keep everything in your user's home,
we'll add an alias for `ejson` that defaults to `~/.ejson/keys`. Adapt the last
command to fit your particular shell setup.

{{< highlight shell >}}
mkdir -p ~/.ejson/keys
alias ejson='ejson --keydir ~/.ejson/keys'
echo "alias ejson='ejson --keydir ~/.ejson/keys'" >> ~/.bash_profile
{{</ highlight >}}

# Using an encrypted `.tfvars` file

First, we need to create a new keypair:

{{< highlight shell >}}
ejson keygen -w
{{</ highlight >}}

Now add the following to a `variables.tfvars.ejson` file, replacing
`__OUTPUT_FROM_EJSON_KEYGEN__` with the public key outputted from the previous
command:

{{< gist mtpereira f18ea33727b05c06c6e9399d3cdd6093 "variables.tfvars.ejson">}}

As you see, this format is immediately usable by Terraform, as it's just a JSON
file, with key-value pairs that hold our Terraform variables.

To encrypt the file, it's as easy as running:

{{< highlight shell >}}
$ ejson encrypt variables.tfvars.ejson
Wrote 211 bytes to variables.tfvars.ejson.
{{</ highlight >}}

This is the file that you should add to your repository to save the variables
safely.

It's also pretty easy to decrypt it:

{{< highlight shell >}}
ejson decrypt variables.tfvars.ejson -o variables.tfvars
{{</ highlight >}}

You should add these `.tfvars` files to your `.gitignore` so that they don't end
up in your repository:

{{< gist mtpereira f18ea33727b05c06c6e9399d3cdd6093 ".gitignore">}}

This will keep the encrypted file intact and create a new, decrypted file.
At this point, you'd be able to run a `terraform plan` as per usual:

{{< highlight shell >}}
terraform plan -var-file variables.tfvars
{{</ highlight >}}

# Streamline it with `make`

We can create a very simple `Makefile` to stream line this process after the
initial setup:

{{< gist mtpereira f18ea33727b05c06c6e9399d3cdd6093 "Makefile" >}}

So now, you just need to run `make plan` and `make apply` to use your safely
encrypted secrets.

# Conclusion

This is just a very simple way to keep your infrastructure's configuration
secrets on a git repository, whilst keeping them readily available for Terraform
to consume. I would use this strategy for simple bootstrapping and infrastructure
provision and use tools such as
[Vault](https://www.vaultproject.io/) for more complex scenarios, for instance,
applications dynamically fetching their configurations on startup.

If you'd like to discuss this post, please drop me a line using [any of my contacts](about).

# Resources
1. [EJSON on Github](https://github.com/Shopify/ejson)

