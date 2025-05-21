# Complete Guide to Creating Multiple Git-Managed Products in AWS Service Catalog

## Step 1: Set Up Your Git Repository Structure

First, create a repository with the following structure:

```
service-catalog-products/
├── README.md
├── products/
│   ├── ec2-instance/
│   │   ├── v1.0/
│   │   │   ├── product.yaml
│   │   │   └── ec2-template.yaml
│   ├── s3-bucket/
│   │   ├── v1.0/
│   │   │   ├── product.yaml
│   │   │   └── s3-template.yaml
└── service-catalog-config.yaml
```

## Step 2: Create Your CloudFormation Templates

### EC2 Instance Template (ec2-template.yaml)
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'EC2 instance template for Service Catalog'
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t3.micro
    Description: EC2 instance type
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Name of an existing EC2 KeyPair
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC where the instance will be created
  SubnetId:
    Type: AWS::EC2::Subnet::Id
    Description: Subnet where the instance will be created
  
Resources:
  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for Service Catalog EC2 instance
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: SC-EC2-SecurityGroup

  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: ami-0c55b159cbfafe1f0  # Amazon Linux 2 AMI (replace with appropriate AMI for your region)
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !GetAtt EC2SecurityGroup.GroupId
      Tags:
        - Key: Name
          Value: ServiceCatalogEC2

Outputs:
  InstanceId:
    Description: The Instance ID
    Value: !Ref MyEC2Instance
  InstancePublicIP:
    Description: Public IP address of the instance
    Value: !GetAtt MyEC2Instance.PublicIp
```

### S3 Bucket Template (s3-template.yaml)
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'S3 bucket template for Service Catalog'
Parameters:
  BucketNamePrefix:
    Type: String
    Default: sc-bucket
    Description: Prefix for the S3 bucket name (will be combined with random string)
  EnableVersioning:
    Type: String
    Default: 'true'
    AllowedValues:
      - 'true'
      - 'false'
    Description: Enable bucket versioning
  
Resources:
  RandomString:
    Type: AWS::CloudFormation::CustomResource
    Properties:
      ServiceToken: !ImportValue RandomStringLambdaArn  # You would need this Lambda function
  
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${BucketNamePrefix}-${RandomString}"
      VersioningConfiguration:
        Status: !If [VersioningEnabled, Enabled, Suspended]
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      Tags:
        - Key: Purpose
          Value: ServiceCatalogProduct

Conditions:
  VersioningEnabled: !Equals [!Ref EnableVersioning, 'true']

Outputs:
  BucketName:
    Description: Name of the created S3 bucket
    Value: !Ref MyS3Bucket
  BucketArn:
    Description: ARN of the S3 bucket
    Value: !GetAtt MyS3Bucket.Arn
```

## Step 3: Create Product Configuration Files

### EC2 Product Config (product.yaml under ec2-instance/v1.0/)
```yaml
Name: EC2-Instance-Standard
Description: Standard EC2 instance deployment with security group
Owner: IT Infrastructure Team
Distributor: Cloud Operations
SupportDescription: For support, contact cloud-ops@example.com
SupportEmail: cloud-ops@example.com
SupportUrl: https://wiki.example.com/cloud/ec2-support
ProvisioningArtifacts:
  - Name: v1.0
    Description: Initial version
    Info:
      LoadTemplateFromURL: ec2-template.yaml
```

### S3 Product Config (product.yaml under s3-bucket/v1.0/)
```yaml
Name: S3-Bucket-Standard
Description: Standard S3 bucket with versioning and encryption
Owner: IT Infrastructure Team
Distributor: Cloud Operations
SupportDescription: For support, contact cloud-ops@example.com
SupportEmail: cloud-ops@example.com
SupportUrl: https://wiki.example.com/cloud/s3-support
ProvisioningArtifacts:
  - Name: v1.0
    Description: Initial version
    Info:
      LoadTemplateFromURL: s3-template.yaml
```

## Step 4: Create the Service Catalog Configuration File (service-catalog-config.yaml)
```yaml
SchemaVersion: 1.0
Portfolio:
  Name: Infrastructure Components
  Description: Standard infrastructure components for applications
  ProviderName: IT Infrastructure
  Associations:
    - PrincipalType: IAM
      PrincipalARN: arn:aws:iam::123456789012:role/ServiceCatalogEndUsers
    - PrincipalType: IAM
      PrincipalARN: arn:aws:iam::123456789012:group/Developers
  Tags:
    - Key: Environment
      Value: Production
Products:
  - ProductFile: products/ec2-instance/v1.0/product.yaml
  - ProductFile: products/s3-bucket/v1.0/product.yaml
```

## Step 5: Create a Connection to Your Git Repository in AWS

1. Go to the AWS Console and navigate to the Developer Tools > CodeStar Connections
2. Click "Create connection"
3. Select your git provider (GitHub, Bitbucket, etc.)
4. Name your connection (e.g., "ServiceCatalogRepoConnection")
5. Follow the steps to authorize AWS with your git provider
6. Complete the connection setup

## Step 6: Create the Service Catalog Products via the AWS Console

1. Navigate to AWS Service Catalog
2. From the navigation, click "Service Catalog" > "Product list"
3. Click "Create multiple git managed products" (as shown in your screenshot)
4. Fill in the form:
   - Common Product details:
     - Owner: "IT Infrastructure Team"
     - (Expand "Product details, support details, and tags" if needed)
   - External repository details:
     - Select "Connect to your provider using AWS CodeStar Connections"
     - Choose the connection you created earlier ("ServiceCatalogRepoConnection")
     - Repository name: "service-catalog-products" (your repository name)
     - Branch: "main" (or your default branch)
     - Path to configuration file: "service-catalog-config.yaml"
5. Click "Create products"

## Step 7: Verify Your Products

1. In the AWS Service Catalog console, check that both products appear in your portfolio
2. Verify that the portfolio has been shared with the appropriate IAM roles/groups
3. Test a product by launching it from the end-user view

## Step 8: Update Your Products (When Needed)

To update your products in the future, simply:
1. Create a new version folder in your repository (e.g., "v1.1")
2. Add the updated template and product files
3. Update the service-catalog-config.yaml to reference the new versions
4. Commit and push your changes

The git-managed products in Service Catalog will automatically update based on your repository changes.

## Best Practices

1. Use a CI/CD pipeline to validate your CloudFormation templates before pushing them to the repository
2. Keep a consistent naming convention for your products and versions
3. Document parameter constraints and descriptions thoroughly
4. Use tags consistently across all resources
5. Consider implementing a pull request review process for changes to the Service Catalog products

This comprehensive setup allows you to manage your AWS resources as code and provides a standardized way for users in your organization to deploy approved resources.