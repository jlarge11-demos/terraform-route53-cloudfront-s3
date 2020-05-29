# Introduction
This project provides a way to quickly stand up an AWS S3 static website by using Terraform with a Terraform Cloud backend.  When everything is created, you will have the following...

* An S3 bucket that contains the static content for your site.
* An CloudFront distribution that sits in front of your S3 bucket.
* An Origin Access Identity that the CloudFront distribution will use to authenticate to your S3 bucket.
* An SSL certificate that gets added to your CloudFront distribution.  This will be validated with DNS.
* A Route53 hosted zone for your site.

# What's in this Repo?
### The hostedzone folder
This folder contains the Terraform config for the DNS hosted zone for your site.  It should be stood up before applying the `site` folder.  **Important:**  When applying, do not run `terraform apply` on its own.  Instead, run `./tfapply` which will also run an AWS CLI command to sync the name servers between the newly created hosted zone and the domain registration.  This is admittedly pretty awkward, which is why I separated this part into its own folder.  While it should be fine to tear down and repave the rest of the infrastructure in this project, the hosted zone should probably be left alone once it's been created, especially since it costs $0.50 every time you create a new one.  For more details about these name server sync issues, I've provided more information further down in this writeup.

### The site folder
This folder contains the Terraform config for the rest of the infrastructure.  You should be able to repave this as many times as you want, but whenever you do, you'll also need to push the static content up again by going to your `main-ui` folder (or your `main-ui` repo if you separated it out) and running `npm run deploy`.

### The main-ui folder
This folder contains a ReactJS UI.  In its current state, it's nothing more than what you generate when you run `npx create-react-app main-ui`.  Typically, you would probably separate this out into its own Git repo, but I'm keeping it with the rest of the project to make the demo a little easier.  Eventually, you'll build this and push the results up into an S3 bucket that gets created from the Terraform config in the `site` folder.

# How are we managing state?
State and variable management is handled by Terraform Cloud in an organization that you create. Environment management (dev, prod, etc) is a little wonky with Terraform Cloud.  Each environment (e.g. `prod`) will be represented by one local workspace named `prod` and two remote workspaces with `prod` combined with the particular infrastructure folder (e.g. `site-prod`).  Currently, the only environment is `prod`, so that means your Terraform Cloud organization will have `hostedzone-prod` and `site-prod`.  If you add a `dev` environment, you would also have `hostedzone-dev` and `site-dev`.   Unfortunately, that means you have to repeat many of the same variables in all of them.  I'm not really sure of a better way to manage this.  Local `tfvars` files will force you to complicate your `terraform` commands, and some of these variables contain secrets.

Currently, each remote environment needs to carry the following variables...

* `aws_access_key_id` for the IAM user that will be authenticating your Terraform run to AWS.
* `aws_secret_access_key` for the IAM user that will be authenticating your Terraform run to AWS.
* `environment` - My original attempt was to avoid having this variable and instead refer `var.workspace` in the config, but that always brings back the value `default` for some reason.  I found an [issue](https://github.com/hashicorp/terraform/issues/22802) out there that gets into that confusion, but it appears to remain unresolved.

# Let's start building your site
Now that we've gone through the contents of this repo and explained our usage of remote state management, it's time to build a site and expose it to the internet.  The following assumptions are made...

* You have an AWS root account setup.
* You have the AWS CLI installed.  You don't necessarily need this to use Terraform in general, but there's an awkward part of these instructions where you run a bash script that runs Terraform and an AWS CLI command.
* You already have an admin level IAM user with an access key and a secret key.  This user will authenticate Terraform to AWS.  In the real world, you'll probably be more sophisticated with your IAM setup, but this should get us by for what we're learning here.
* You have AWS CLI v1.8.56 or higher.
* Your access key and secret key for your admin user are stored in `~/.aws/credentials`.
* You have Terraform CLI v0.12.24 or higher.

### Coming up with your domain name
This can be anything you like, but if you're just playing around, one thing you can do is to visit [this site](https://frightanic.com/goodies_content/docker-names.php) which will generate one of those funny names that Docker gives you.  The last time I ran this, I got `berserk_nobel`, so for this writeup, I'm going to go with a domain of `berserknobel.com`.  For the rest of these instructions, I'm going to reference `berserknobel` simply because I'd rather do that than put ugly placeholders all over the place.  When you execute these instructions, you should simply replace `berserknobel` with the site name that you're going with.

### Registering on Route53 in the AWS Console
Unlike most of our AWS infrastructure, we're going to do this manually.  This will cost you $12 if you're using a `.com` domain (or $282 for a `.sucks` domain, holy smokes).  Alternatively, if you already have a domain in Route53 for playing around with, you can simply use that one and skip this section.  If you decide to register your own domain in Route53, then you'll need to do the following...

1. Login to the AWS console as your admin IAM user.
2. Navigate to Route53.
3. Under "Register Domain", type `berserknobel` and choose `.com` from the dropdown.
4. Click "Check".
5. If it's available, then click "Add to Cart" and then "Continue".
6. Accept the defaults and click "Continue".
7. Accept the terms and click on "Complete Order".

This can take up to three days, but the last time I did this, it took about an hour.  If you try to navigate to that domain earlier than that, your browser will probably return some sort of certificate error.

### Terraform Cloud Setup
1. Navigate to [Terraform Cloud](https://app.terraform.io/app).
2. Under "Choose an organization", click on "Create new organization".
3. For Organization Name, name it after your domain.  For this writeup, I named mine `berserknobel`.
4. Provide your email address and click "Create Organization".
5. You'll be asked to connect to a Version Control Provider.  Just click on "No VCS Connection".  I've not explored this part myself.
6. You'll be asked to create a workspace, but just cancel out of this.  We'll take care of this in the next section.

### Creating the hostedzone-prod workspace
1. Go back to the main page for your organization.
2. Click on "New Workspace".
3. For "Workspace Name", enter `hostedzone-prod`.
4. Click "Create Workspace".  You'll be taken to the main page of that workspace, and it will say that it's waiting for configuration.  That's fine.  We'll take care of that later.
5. Click on "Variables", and add the following...
   * `aws_access_key_id` should be set to your IAM user's access key and marked as secret.
   * `aws_secret_access_key` should be set to your IAM user's secret key and marked as secret.
   * `environment` should be set to `prod`.  My original attempt was to avoid having this variable and instead refer `var.workspace` in the config, but that always brings back the value `default` for some reason.  I found an [issue](https://github.com/hashicorp/terraform/issues/22802) out there that gets into that confusion, but it appears to remain unresolved.

### Creating the site-prod workspace
Follow all of the same steps you took in the previous section when you created the `hostedzone-prod` workspace.

### Forking this repo
At this point, you'll probably want to fork this repo and potentially rename it to something that resembles the domain name you chose.  Also, if you're doing anything more long term, you might also want to take the `main-ui` folder and break that out into its own Git repo.

### Changing your code to use your domain
Various places in the code base reference `__yoursitehere__`.  You'll need to go through the files that contain this and change it to your domain (`berserknobel` in my case).  For convenience, I included commands specific to your OS that you can use, but if these give you problems, you can also do this by hand.  Here are the files that need to be changed...

* `hostedzone/tfapply`
* `hostedzone/setup.tf`
* `site/setup.tf`
* `main-ui/package.json`

#### In Linux
```bash
egrep -lRZ '__yoursitehere__' . | xargs -0 -l sed -i -e 's/__yoursitehere__/berserknobel/g'
```

#### In iOS
```bash
find . -type f | xargs sed -i '' 's/__yoursitehere__/berserknobel/g'
```

### Building with Terraform
#### In the `hostedzone` folder
1. Run `terraform init`.  You'll be prompted to choose a workspace.  The only option right now is `prod`, so choose that.
2. Run `./tfapply` and say `yes` when it prompts for confirmation.  It's important that you don't run `terraform apply` on its own.  The `tfapply` script does some name server syncing that is explained further down in this writeup.

#### In the `site` folder
1. Run `terraform init`.  You'll be prompted to choose a workspace.  The only option right now is `prod`, so choose that.
2. Run `terraform apply` and say `yes` when it prompts for confirmation.  The creation of the CloudFront distribution takes a while.  The last time I ran this apply, it took around 10 minutes.

#### In the `main-ui` folder
1. Run `npm install`.
2. Run `npm run deploy`.  This will do a build and then push the contents up to your newly created S3 bucket.

#### Trying it out
1. Wait about an hour for your DNS and certificate to be fully propagated.  The wait time varies, and I don't know enough to say why.
2. Visit `berserknobel.com` on your browser.  It should open up with a generic ReactJS page.

# Tearing Down
Tearing everything down should be simple.  You just run `terraform destroy` from your `site` folder and then do the same from your `hostedzone` folder.  If you're really done with everything, you might also want to go into Terraform Cloud and remove your organization as well.

# Problems with name servers
By far, my biggest hurdle in getting all of this to work is that every time the DNS hosted zone is recreated, four new name servers are randomly assigned to it, and these will not be the same servers that are in the domain registration.  It took me a day of troubleshooting before I realized this, mainly because I'm not experienced in troubleshooting DNS issues (I'm still pretty bad at it).  Once I saw this as the issue, I tried the following things...

### Attempt 1:  Go into the AWS console and manually update the name servers
This solved my problem right away, but I didn't want to stay with a manual solution very long.

### Attempt 2:  See if there is a Terraform resource for the domain registration
I did see a somewhat popular [issue](https://github.com/hashicorp/terraform/issues/5368) where somebody asked for a resource called `aws_route53_domain`, but from the looks of it, it kind of died on the vine.

### Attempt 3:  Add a local-exec provisioner with a script that syncs the name servers
I think I could get this to work if I was doing local Terraform, but I ran into problems with Terraform Cloud because its worker servers don't have much software on them.  In particular, they don't have the AWS CLI.  I went down this road for a while but ultimately gave up on it.

### Attempt 4:  Create a separate script that runs terraform apply and then syncs up the name servers
This is what I went with.  It works, and it's better than dealing with things manually, but it's not my favorite thing in the world.  First of all, it takes that name server syncing out of Terraform, so technically you could argue that it's a form of configuration drift.  Second, it's real easy for somebody to forget about this script and simply run `terraform apply`.  I mitigate this somewhat by placing this stuff in its own folder and advising that we don't touch it once it's created.
