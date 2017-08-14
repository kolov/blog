# From Zero to Continuous Delivery to Kubernetes on AWS in 2 hours

For a developer, Kubernetes is a dream platform to develop for. 
Especially in the microservices world, it is great to have acces to its simple powerful API. 
Setting up a Kubernetes cluster is a different story, or at least used to be. 
In my experience, any other platform than Google Container Engine required too much operational effort to set up and maintain. 

Some time ago I had set up a small cluster on GKE with CircleCI deploying live. 
<a href="https://github.com/kubernetes/kops">Kops</a> claims to make using AWS as simple as GKE, 
so this weekend I decided to move a GKE cluster, including configurati and applications to AWS. 

## Kubernetes on AWS
First, create the cluster. Coosing the simplest DNS option, a fully operation cluster can be set up with just 
the following commands:

      aws s3api create-bucket --bucket kolov-k8s1-state-store --region us-east-1
      aws s3api put-bucket-versioning --bucket kolov-k8s1-state-store --versioning-configuration Status=Enabled
      export KOPS_STATE_STORE=s3://kolov-k8s1-state-store 
      kops create cluster --zones eu-central-1a  kolov-k8s1.k8s.local

The commands above are more of an example to show how simple it is to create a cluster with `kops`, for the details 
follow the official
 instructions.
 OK, now we have a cluster, and we clould start manually rinning `kubectl` to run our application there, but 
doing that manually is not cool. 

## Container Registry
As we are going to build an deploy containers, we need a container registry. 
If you have one setup somwewhere - great. Setting up one on AWS is pretty straightforward - go to `EC2 Container 
Service` and create the repository. The build process need push access to that ragistry - create a IAM user with 
`arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser` policy. Note its Access Key ID and Secret Key for use by 
the build process.

## CircleCI

There mare certainly many cjoices for a CI server, but we want to be redy in one houre - go directly to CircleCI. They
offer a free 1500 hours/month plan, isn't that insane? After you have a account, select a project to build. A file 
named `circleci.yml` defined the build:

    machine:
      services:
        - docker
      environment:
        APP_NAME: curriculi-docs-service
        BUILD_TARGET_DIR: .
    
    dependencies:
      cache_directories:
          - "~/.ivy2"
          - "~/.sbt"
      override:
        - sbt universal:packageZipTarball
        - wget https://raw.githubusercontent.com/kolov/k8s-stuff/master/circleci/deploy-aws.sh
        - chmod +x deploy-aws.sh
        - ./deploy-aws.sh
        
## The build script