# Amazon S3 — Practical Assignments

## Assignment 1 — S3 Bucket Setup and File Operations

```bash
# 1. Create a bucket (globally unique name!)
BUCKET="aws-notes-practice-$(date +%s)"
aws s3api create-bucket --bucket $BUCKET --region us-east-1
echo "Bucket: $BUCKET"

# 2. Create some test files
echo "Hello AWS S3! Created: $(date)" > test.txt
echo '{"user":"manoj","role":"aws-learner"}' > config.json
dd if=/dev/urandom bs=1K count=100 | base64 > large-file.txt  # ~100KB file

# 3. Upload individual files
aws s3 cp test.txt s3://$BUCKET/
aws s3 cp config.json s3://$BUCKET/configs/config.json
aws s3 cp large-file.txt s3://$BUCKET/data/large-file.txt --storage-class STANDARD_IA

# 4. Upload directory
mkdir -p website/images
echo "<h1>My S3 Website</h1>" > website/index.html
echo "<h1>Error 404</h1>" > website/error.html
echo "img placeholder" > website/images/logo.txt

aws s3 sync ./website/ s3://$BUCKET/website/

# 5. List all objects
aws s3 ls s3://$BUCKET/ --recursive --human-readable

# 6. Get object details
aws s3api head-object --bucket $BUCKET --key test.txt

# 7. Generate presigned URL (valid 1 hour)
aws s3 presign s3://$BUCKET/test.txt --expires-in 3600

# 8. Copy object within S3
aws s3 cp s3://$BUCKET/test.txt s3://$BUCKET/backups/test-backup.txt

# 9. Download
aws s3 cp s3://$BUCKET/configs/config.json ./downloaded-config.json
cat downloaded-config.json

# 10. Cleanup
rm test.txt config.json large-file.txt downloaded-config.json
rm -rf website
```

---

## Assignment 2 — Versioning and Recovery

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# Upload version 1
echo "Version 1 - original content" > myfile.txt
aws s3 cp myfile.txt s3://$BUCKET/myfile.txt
V1=$(aws s3api list-object-versions --bucket $BUCKET --prefix myfile.txt \
  --query 'Versions[0].VersionId' --output text)
echo "Version 1 ID: $V1"

# Upload version 2
echo "Version 2 - updated content" > myfile.txt
aws s3 cp myfile.txt s3://$BUCKET/myfile.txt
V2=$(aws s3api list-object-versions --bucket $BUCKET --prefix myfile.txt \
  --query 'Versions[0].VersionId' --output text)
echo "Version 2 ID: $V2"

# "Accidentally" delete the file
aws s3 rm s3://$BUCKET/myfile.txt

# Try to access - file appears gone
aws s3 ls s3://$BUCKET/myfile.txt  # Nothing shown

# But all versions still exist!
aws s3api list-object-versions --bucket $BUCKET --prefix myfile.txt --output table

# Restore version 1 by copying it back
aws s3api copy-object \
  --bucket $BUCKET \
  --copy-source "$BUCKET/myfile.txt?versionId=$V1" \
  --key myfile.txt

aws s3 cp s3://$BUCKET/myfile.txt -
# Prints: "Version 1 - original content" — restored!
```

---

## Assignment 3 — Static Website Hosting

```bash
# Enable static website hosting
aws s3api delete-public-access-block --bucket $BUCKET

aws s3api put-bucket-website \
  --bucket $BUCKET \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'

# Add bucket policy for public read
aws s3api put-bucket-policy --bucket $BUCKET --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::'$BUCKET'/*"
  }]
}'

# Create website files
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>My S3 Site</title></head>
<body style="font-family:Arial;padding:40px;background:#f5f5f5">
  <h1>Hello from Amazon S3!</h1>
  <p>This website is hosted on S3 — no servers needed!</p>
  <ul>
    <li>Static HTML/CSS/JS only</li>
    <li>Infinitely scalable</li>
    <li>Costs pennies per month</li>
  </ul>
</body>
</html>
EOF

cat > error.html << 'EOF'
<html><body><h1>404 - Page Not Found</h1><a href="/">Go Home</a></body></html>
EOF

aws s3 cp index.html s3://$BUCKET/ --content-type text/html
aws s3 cp error.html s3://$BUCKET/ --content-type text/html

REGION=$(aws configure get region)
echo "Website URL: http://$BUCKET.s3-website-$REGION.amazonaws.com"
curl http://$BUCKET.s3-website-$REGION.amazonaws.com
```

---

## Assignment 4 — Lifecycle Policy

```bash
# Create lifecycle policy: auto-archive logs
aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "log-archive",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Transitions": [
          {"Days": 30, "StorageClass": "STANDARD_IA"},
          {"Days": 90, "StorageClass": "GLACIER"}
        ],
        "Expiration": {"Days": 365}
      },
      {
        "ID": "delete-old-versions",
        "Status": "Enabled",
        "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
      }
    ]
  }'

aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET --output table

# Upload logs to test prefix
for i in {1..10}; do
  echo "Log entry $i - $(date)" | aws s3 cp - s3://$BUCKET/logs/log-$i.txt
done
aws s3 ls s3://$BUCKET/logs/ --human-readable
```

---

## Clean Up

```bash
# Remove all versions
aws s3api list-object-versions --bucket $BUCKET \
  --query 'Versions[*].[Key,VersionId]' --output text | \
  while read KEY VID; do
    aws s3api delete-object --bucket $BUCKET --key "$KEY" --version-id "$VID"
  done

# Remove delete markers
aws s3api list-object-versions --bucket $BUCKET \
  --query 'DeleteMarkers[*].[Key,VersionId]' --output text | \
  while read KEY VID; do
    aws s3api delete-object --bucket $BUCKET --key "$KEY" --version-id "$VID"
  done

# Delete bucket
aws s3api delete-bucket --bucket $BUCKET
echo "Cleaned up!"
```

---

## Interview Practice

### Scenario: Static Website with CloudFront
> "Your company wants to host a global marketing website with sub-100ms load times for users worldwide. They don't want to manage servers. What do you recommend?"

**Answer**: S3 + CloudFront:
1. Upload static HTML/CSS/JS/images to S3
2. Create CloudFront distribution pointing to S3 as origin
3. CloudFront caches content at 600+ edge locations globally
4. Users hit nearest edge location → sub-50ms latency
5. Add Route 53 for custom domain with HTTPS (ACM certificate)
6. Enable S3 versioning for rollback capability
7. Total server management: zero. Cost: pennies per month.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
