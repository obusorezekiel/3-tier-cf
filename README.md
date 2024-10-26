# AWS Infrastructure Stack

This CloudFormation template provisions a complete AWS infrastructure stack for running containerized applications with high availability and scalability.

## Architecture Overview

The stack creates the following architecture:

- **VPC** with 6 subnets across 2 AZs:
  - 2 Public subnets (for ALB)
  - 2 Private subnets (for ECS tasks)
  - 2 Private subnets (for RDS)
- **ECS Fargate** cluster with auto-scaling
- **Application Load Balancer** in public subnets
- **RDS PostgreSQL** instance in private subnets
- **ECR Repository** for container images
- **NAT Gateway** for private subnet internet access
- Comprehensive security groups and IAM roles

## Prerequisites

- AWS CLI installed and configured
- Necessary AWS permissions to create all resources
- Docker installed (for pushing images to ECR)

## Parameters

| Parameter | Description | Constraints |
|-----------|-------------|-------------|
| EnvironmentName | Environment name (e.g., dev, prod) | Default: dev |
| DBPassword | Database admin password | 8-40 chars, allowed special chars: !@#$%^&*()_+= |
| DBUsername | Database username | - |
| ImageTag | Docker image tag to deploy | - |

## Environment-specific Features

The template includes conditional logic that enables different features based on the environment:

### Production Environment
- Multi-AZ RDS deployment
- 7-day backup retention
- Deletion protection enabled

### Development Environment
- Single-AZ RDS deployment
- 1-day backup retention
- Deletion protection disabled

## Security Features

- Private subnets for application and database layers
- Security groups with minimal required access:
  - ALB: Inbound 80 from anywhere
  - ECS: Inbound 80 from ALB only
  - RDS: Inbound 5432 from ECS only
- NAT Gateway for secure outbound internet access
- No direct database internet accessibility

## Auto Scaling Configuration

- ECS Service will auto-scale based on CPU utilization
- Target tracking scaling policy:
  - Target CPU utilization: 75%
  - Min capacity: 1
  - Max capacity: 10
  - Scale in/out cooldown: 60 seconds

## Networking Layout

| Subnet | CIDR Block | Purpose |
|--------|------------|---------|
| Public 1 | 10.0.1.0/24 | ALB, NAT Gateway |
| Public 2 | 10.0.2.0/24 | ALB |
| Private 1 | 10.0.3.0/24 | ECS Tasks |
| Private 2 | 10.0.4.0/24 | ECS Tasks |
| Private 3 | 10.0.5.0/24 | RDS |
| Private 4 | 10.0.6.0/24 | RDS |

## Deployment Instructions

1. Create an ECR repository:
```bash
aws cloudformation create-stack \
  --stack-name <stack-name> \
  --template-body file://template.yml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=<env> \
    ParameterKey=DBPassword,ParameterValue=<password> \
    ParameterKey=DBUsername,ParameterValue=<username> \
    ParameterKey=ImageTag,ParameterValue=<tag>
```

2. Push your Docker image:
```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker tag <image>:<tag> <account-id>.dkr.ecr.<region>.amazonaws.com/<env>-repo:<tag>
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/<env>-repo:<tag>
```

## Outputs

| Output | Description |
|--------|-------------|
| LoadBalancerDNS | DNS name of the Application Load Balancer |
| RDSInstanceEndpoint | Endpoint for the RDS instance |
| ECRRepositoryURI | URI of the ECR repository |

## Monitoring and Logging

- ECS task logs are sent to CloudWatch Logs
  - Log group: `/ecs/<environment-name>`
  - Retention: 30 days
- RDS logs and metrics available in CloudWatch
- ALB access logs can be enabled (not included in template)

## Cost Optimization

- NAT Gateway shared across all private subnets
- T3.micro instance type for RDS in non-production
- Fargate tasks with minimal CPU/memory allocation (256 CPU units, 512MB RAM)

## Cleanup

To delete the stack:
```bash
aws cloudformation delete-stack --stack-name <stack-name>
```

**Note**: If deletion protection is enabled (production environment), you'll need to manually disable it for the RDS instance before deleting the stack.

## Contributing

When modifying this template:
1. Always update the README with any new parameters or resources
2. Test changes in a development environment first
3. Validate the template using `aws cloudformation validate-template`
4. Consider the impact on existing deployments

## Security Considerations

- Regularly rotate the database password
- Monitor CloudWatch logs for suspicious activity
- Review security group rules periodically
- Keep container images updated with security patches
- Consider enabling AWS Config rules for compliance monitoring

## Support

For issues or questions:
1. Check CloudWatch logs for application issues
2. Review CloudFormation events for deployment issues
3. Verify security group rules if experiencing connectivity issues
4. Ensure IAM roles have appropriate permissions