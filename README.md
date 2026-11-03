# AWS CloudFormation Windows .NET Deployment

This repository contains an ASP.NET Core MVC application and an AWS
CloudFormation template for provisioning a Windows Server 2022 EC2 instance
with IIS. The template creates the network and server infrastructure; it does
not currently publish or install the ASP.NET Core application onto that server.

## Solution overview

```text
Browser
  |
  | HTTP :80
  v
EC2 Windows Server 2022 (public subnet)
  |
  +-- IIS, installed by EC2 user data
  +-- Placeholder page: "AWS Windows Server IIS is Ready"
  +-- SSM managed instance role

ASP.NET Core MVC application
  +-- Separate project in application/WindowsWebApp
  +-- Can be run locally with the .NET 10 SDK
  +-- Not yet deployed to the EC2 instance by the supplied template
```

### Application

`application/WindowsWebApp` is an ASP.NET Core MVC application targeting
.NET 10. It includes Home and Privacy pages, static assets, and the standard
development/production error handling setup.

Run it locally from the repository root:

```powershell
dotnet restore .\application\WindowsWebApp\WindowsWebApp.csproj
dotnet run --project .\application\WindowsWebApp\WindowsWebApp.csproj
```

Use the local URL printed by `dotnet run` (the configured HTTP URL is
`http://localhost:5062`).

### AWS infrastructure

`infrastructure/cloudformation/main.yaml` provisions:

- A VPC (`10.10.0.0/16`), a public subnet, internet gateway, and public route.
- A security group allowing HTTP on port 80 from the `AllowedHttpCidr`
  parameter.
- A Windows Server 2022 EC2 instance using the current AWS-published Windows
  Server 2022 AMI resolved through Systems Manager Parameter Store.
- An EC2 instance profile with `AmazonSSMManagedInstanceCore`.
- EC2 user data that installs IIS and writes a placeholder page to
  `C:\inetpub\wwwroot\index.html`.

The template parameters are:

| Parameter | Default | Description |
| --- | --- | --- |
| `InstanceType` | `t3.small` | EC2 instance type |
| `AllowedHttpCidr` | `0.0.0.0/0` | CIDR allowed to access HTTP |

## Deploy the infrastructure

### Prerequisites

- An AWS account and credentials configured for the AWS CLI.
- AWS CLI v2 and permission to create CloudFormation, EC2, VPC, and IAM
  resources.

From the repository root, replace the example CIDR with the public IP address
that should access the site (use `/32` to allow a single IPv4 address):

```powershell
aws cloudformation deploy `
  --template-file .\infrastructure\cloudformation\main.yaml `
  --stack-name windows-dotnet-demo `
  --parameter-overrides InstanceType=t3.small AllowedHttpCidr=203.0.113.10/32 `
  --capabilities CAPABILITY_IAM
```

After the stack completes, retrieve its outputs:

```powershell
aws cloudformation describe-stacks `
  --stack-name windows-dotnet-demo `
  --query "Stacks[0].Outputs"
```

Open the `WebsiteURL` output in a browser to see the IIS placeholder page.
The stack also outputs the EC2 instance ID, public IP, VPC ID, and subnet ID.

Remove the resources when they are no longer needed:

```powershell
aws cloudformation delete-stack --stack-name windows-dotnet-demo
```

## Current scope and security notes

- The infrastructure currently serves only the placeholder IIS page. The MVC
  application is not copied to the instance, its .NET runtime is not installed
  by the template, and IIS is not configured to host it.
- The files under `.github/workflows` and `infrastructure/scripts` are
  placeholders; automated build/deployment workflows and application
  deployment scripts are not implemented yet.
- The template defaults `AllowedHttpCidr` to `0.0.0.0/0`, making the HTTP site
  publicly reachable. Restrict this parameter to trusted source addresses for
  anything beyond a disposable demo.
- The template exposes HTTP only; it does not configure HTTPS or a domain name.
- EC2 and related AWS resources may incur charges while the stack exists.