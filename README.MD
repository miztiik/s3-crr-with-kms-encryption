# S3 Cross Region Replication with KMS Encrypted Objects

By default, Amazon S3 doesn't replicate objects that are stored at rest using server-side encryption with AWS KMS-managed keys. To replicate encrypted objects, you modify the bucket replication configuration to tell Amazon S3 to replicate these objects. 

This demonstration explains how to use the AWS CLI to change the bucket replication configuration to enable replicating encrypted objects.

![Fig : S3 Cross Region Replication with Encrypted Objects](https://raw.githubusercontent.com/miztiik/s3-crr-with-kms-encryption/master/images/s3-crr-kms-encryption.png)

Although this demo is written for same account, by changing the `src_profile` and `dest_profile` along with appropriate KMS `*_key_arn` you should be able implement the same across accounts

#### Follow this article in [Youtube](https://youtu.be/d8bvvGEJU68)
1. ## Source & Destination Bucket configurations
   Let us begin by setting up some global variables that we will be using through out this demo.
   - ### Source Bucket Information
      ```sh
      src_bkt="src-bkt-101"
      src_bkt_region="us-east-1"
      src_profile="default"
      src_kms_arn="arn:aws:kms:us-east-1:589512693537:key/7c711927-4cb0-4aa7-8e2e-81ec63602dd7"
      src_kms_region="us-east-1"
      ```
   - ### Destination Bucket Information
      ```sh
      dest_bkt="dest-bkt-101"
      dest_bkt_region="us-east-2"
      dest_profile="default"
      dest_kms_arn="arn:aws:kms:us-east-2:589512693537:key/91b7d8e5-e9ad-4d82-95ba-2bb0c43ed9cc"
      dest_kms_region="us-east-2"
      ```
   - ### Generic Configs
      ```sh
      iam_role_name="s3-crr-with-kms-role"
      trust_policy="./s3-crr-with-kms-role-trust-policy.json"
      permissions_policy="./s3-crr-with-kms-role-permissions-policy.json"
      bkt_replication_config="./s3-crr-config.json"
      ```

1. ## Create Source Bucket
   Lets create the source & destinations buckets in different regions & enable versioning
   ```sh
   aws s3api create-bucket \
      --bucket ${src_bkt} \
      --region ${src_bkt_region} \
      --profile ${src_profile}

   ##### Because of S3 constraints for virginia, do not specify region for this region
   ##### https://docs.aws.amazon.com/AmazonS3/latest/API/RESTBucketPUT.html 
   ```

   - ### Enable Versioning on Source Bucket
      ```sh
      aws s3api put-bucket-versioning \
         --bucket ${src_bkt} \
         --versioning-configuration Status=Enabled \
         --profile ${src_profile}
      ```

1. #### Create Destination Bucket
   ```sh
   aws s3api create-bucket \
      --bucket ${dest_bkt} \
      --region ${dest_bkt_region} \
      --create-bucket-configuration LocationConstraint=${dest_bkt_region} \
      --profile ${dest_profile}
   ```

   - #### Enable Versioning on Destination Bucket
      ```sh
      aws s3api put-bucket-versioning \
         --bucket ${dest_bkt} \
         --versioning-configuration Status=Enabled \
         --profile ${dest_profile}
      ```

1. ## Creating IAM Trust, Permissions Policy & Role
   Before creating the role, we need a trust policy, Role Permissions.
   - ### Trust Policy
      ```sh
      cat > ${trust_policy} << "EOF"
      {
         "Version":"2012-10-17",
         "Statement":[
            {
               "Effect":"Allow",
               "Principal":{
                  "Service":"s3.amazonaws.com"
               },
               "Action":"sts:AssumeRole"
            }
         ]
      }
      EOF
      ```

   - ### IAM Permissions Policy
      
      ```sh
      echo '{
         "Version":"2012-10-17",
         "Statement":[
            {
               "Action":[
                  "s3:ListBucket",
                  "s3:GetReplicationConfiguration",
                  "s3:GetObjectVersionForReplication",
                  "s3:GetObjectVersionAcl"
               ],
               "Effect":"Allow",
               "Resource":[
                  "arn:aws:s3:::'${src_bkt}'",
                  "arn:aws:s3:::'${src_bkt}'/*"
               ]
            },
            {
               "Action":[
                  "s3:ReplicateObject",
                  "s3:ReplicateDelete",
                  "s3:ReplicateTags",
                  "s3:GetObjectVersionTagging"
               ],
               "Effect":"Allow",
               "Condition":{
                  "StringLikeIfExists":{
                     "s3:x-amz-server-side-encryption":[
                        "aws:kms",
                        "AES256"
                     ],
                     "s3:x-amz-server-side-encryption-aws-kms-key-id":[
                        "'${dest_kms_arn}'"
                     ]
                  }
               },
               "Resource":"arn:aws:s3:::'${dest_bkt}'/*"
            },
            {
               "Action":[
                  "kms:Decrypt"
               ],
               "Effect":"Allow",
               "Condition":{
                  "StringLike":{
                     "kms:ViaService":"s3.'${src_kms_region}'.amazonaws.com",
                     "kms:EncryptionContext:aws:s3:arn":[
                        "arn:aws:s3:::'${src_bkt}'/*"
                     ]
                  }
               },
               "Resource":[
                  "'${src_kms_arn}'"
               ]
            },
            {
               "Action":[
                  "kms:Encrypt"
               ],
               "Effect":"Allow",
               "Condition":{
                  "StringLike":{
                     "kms:ViaService":"s3.'${dest_kms_region}'.amazonaws.com",
                     "kms:EncryptionContext:aws:s3:arn":[
                        "arn:aws:s3:::'${dest_bkt}'/*"
                     ]
                  }
               },
               "Resource":[
                  "'${dest_kms_arn}'"
               ]
            }
         ]
      }
      ' | sudo tee ${permissions_policy}
      ```

   - ### Create IAM Role
      We are going to create the role with the correct trust permissions and store the IAM Role ARN for future use.
      ```sh
      iam_role_arn=$(aws iam create-role \
                      --role-name ${iam_role_name} \
                      --assume-role-policy-document file://${trust_policy} \
                      --profile ${src_profile} \
                      --output text \
                      --query 'Role.Arn')
      ```

   - ### Attach IAM Permissions to Role
      Attach the previously created permissions to the newly created role
      ```sh
      aws iam put-role-policy \
          --role-name ${iam_role_name} \
          --policy-document file://${permissions_policy} \
          --policy-name "s3-crr-with-kms-role-permissions-policy" \
          --profile ${src_profile}
      ```

1. ## Configure Replication in Source Bucket
   Now we are ready to implement the replication
   ```sh
   echo '{
      "Role":"'${iam_role_arn}'",
      "Rules":[
         {
            "Status":"Enabled",
            "Prefix": "",
            "Destination":{
               "Bucket":"arn:aws:s3:::'${dest_bkt}'",
               "EncryptionConfiguration":{
                  "ReplicaKmsKeyID":"'${dest_kms_arn}'"
               }
            },
            "SourceSelectionCriteria":{
               "SseKmsEncryptedObjects":{
                  "Status":"Enabled"
               }
            }
         }
      ]
   }
   ' | sudo tee ${bkt_replication_config}
   ```

   - ## Apply Replication configuration to Source Bucket.
      ```sh
      aws s3api put-bucket-replication \
         --replication-configuration file://${bkt_replication_config} \
         --bucket ${src_bkt} \
         --profile ${src_profile}
      ```
1. ## Test the setup 
   1. In the `src_bkt` bucket.Add sample objects. Be sure to choose the encryption option and specify your KMS key to encrypt the objects.
   1. Verify that the `dest_bkt` bucket contains the object replicas and that they are encrypted using the KMS encryption key that you specified in the configuration.
