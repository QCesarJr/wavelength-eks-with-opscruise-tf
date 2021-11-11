WavelengthCloudFormation README

These commands can be run either through the aws cli (download here: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) or by uploading the cloud formation script into the AWS CloudFormation UI and providing the Parameters.

#Create AWSQS Custom Resource Provider Execution Roles for Helm and EKS
aws cloudformation create-stack \
    --stack-name "AWSQS-Helm-EKS-ResourceProvider-Roles" \
    --template-url https://wavelength-cloudformation-templates.s3.us-west-2.amazonaws.com/awsqs-helm-eks-extensionRoles.yaml \
    --capabilities CAPABILITY_NAMED_IAM

#Set EKS Execution Role ARN to variable
export AWSQS_EKSRole=$(aws cloudformation describe-stacks \
        --stack-name "AWSQS-Helm-EKS-ResourceProvider-Roles" \
        --query "Stacks[0].Outputs[?OutputKey=='AWSQSEKSExecutionRoleArn'].OutputValue" --output text)

#Set Helm Execution Role ARN to variable
export AWSQS_HelmRole=$(aws cloudformation describe-stacks \
        --stack-name "AWSQS-Helm-EKS-ResourceProvider-Roles" \
        --query "Stacks[0].Outputs[?OutputKey=='AWSQSHelmExecutionRoleArn'].OutputValue" --output text)

# Activate AWSQS::EKS::Cluster Resource-Type
## Only need to run once per region
aws cloudformation activate-type \
    --public-type-arn "arn:aws:cloudformation:$(aws configure get region)::type/resource/408988dff9e863704bcc72e7e13f8d645cee8311/AWSQS-EKS-Cluster" \
    --execution-role-arn $AWSQS_EKSRole

# Activate AWSQS::Kubernetes::Helm Resource-Type
## Only need to run once per region
aws cloudformation activate-type \
    --public-type-arn "arn:aws:cloudformation:$(aws configure get region)::type/resource/408988dff9e863704bcc72e7e13f8d645cee8311/AWSQS-Kubernetes-Helm" \
    --execution-role-arn $AWSQS_HelmRole

#Deploy EKS Cluster with Worker Nodes in Wavelength Zone
aws cloudformation create-stack \
    --stack-name wl-eks-nov2-1206 \
    --template-url https://wavelength-cloudformation-templates.s3.us-west-2.amazonaws.com/wavelength-eksCluster-withOpsCruise.yaml \
    --disable-rollback \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
        ParameterKey=OpsCruiseValuesURL,ParameterValue="s3://wavelength-cloudformation-templates/cesarquintana-vzpayrch-opscruise-values.yaml" \
        ParameterKey=WavelengthZoneGeo,ParameterValue=us-west-2-wl1-las-wlz-1 \
        ParameterKey=ParentRegionGeo,ParameterValue=us-west-2a \
        ParameterKey=ParentRegion2Geo,ParameterValue=us-west-2b \
        ParameterKey=HelmExecutionRole,ParameterValue="$AWSQS_HelmRole"